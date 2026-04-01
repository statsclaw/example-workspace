# Implementation Specification: interflex Python Package

## 1. Notation

| Symbol | Type | Description |
|--------|------|-------------|
| $Y$ | string | Column name for outcome variable |
| $D$ | string | Column name for treatment variable |
| $X$ | string | Column name for moderator variable |
| $Z$ | list[str] or None | Column names for covariates |
| $FE$ | list[str] or None | Column names for fixed effect variables |
| $IV$ | list[str] or None | Column names for instrumental variables |
| $n$ | int | Number of observations |
| $p$ | int | Number of regressors (excluding FE) |
| $k$ | int | Number of evaluation points (neval) |
| $\hat\beta$ | ndarray(p,) | Estimated coefficient vector |
| $\hat{V}$ | ndarray(p,p) | Estimated variance-covariance matrix of $\hat\beta$ |
| $x$ | float | A single evaluation point for the moderator |
| $X_{eval}$ | ndarray(k,) | Grid of evaluation points |
| $D_{ref}$ | float or list[float] | Reference value(s) for continuous treatment |
| $Z_{ref}$ | ndarray or None | Reference values for covariates |
| $w$ | ndarray(n,) | Observation weights (default: all 1s) |
| $M$ | int | Number of clusters (for clustered SE) |
| $K$ | int | Model rank (number of estimated coefficients) |

## 2. Package Structure

```
interflex/
    __init__.py          # Public API: interflex(), InterflexResult
    core.py              # interflex() entry point, input validation
    linear.py            # interflex_linear() — model fitting, TE/ME computation
    effects.py           # gen_general_te(), gen_te(), gen_te_fe() — treatment/marginal effects
    variance.py          # gen_delta_te(), simulation variance, bootstrap variance
    vcov.py              # vcov_cluster(), robust_vcov(), pcse_vcov()
    uniform.py           # calculate_uniform_quantiles(), calculate_delta_uniform_ci()
    fwl.py               # fwl_demean(), iv_fwl() — Frisch-Waugh-Lovell
    plotting.py          # plot_interflex(), plot_interflex_pool(), plot_predict()
    predict.py           # predict_interflex()
    result.py            # InterflexResult dataclass
    _typing.py           # Type aliases
tests/
    conftest.py          # Fixtures, synthetic data generators
    test_core.py         # Input validation tests
    test_linear.py       # Linear estimator integration tests
    test_effects.py      # TE/ME computation tests
    test_variance.py     # Delta, simulation, bootstrap variance tests
    test_vcov.py         # Clustered, robust, PCSE SE tests
    test_fwl.py          # FWL demeaning tests
    test_uniform.py      # Uniform CI tests
    test_ate.py          # ATE/AME tests
    test_plotting.py     # Plot smoke tests
    test_predict.py      # Predict method tests
pyproject.toml
README.md
```

## 3. pyproject.toml

```toml
[build-system]
requires = ["setuptools>=68.0", "wheel"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "interflex"
version = "0.1.0"
description = "Python implementation of the interflex linear estimator for interaction effects"
requires-python = ">=3.10"
dependencies = [
    "numpy>=1.24",
    "scipy>=1.10",
    "statsmodels>=0.14",
    "pandas>=2.0",
    "matplotlib>=3.7",
    "seaborn>=0.12",
]

[project.optional-dependencies]
dev = ["pytest>=7.0", "pytest-cov"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

## 4. Module Specifications

### 4.1 `_typing.py` — Type Aliases

```python
from __future__ import annotations
from typing import Literal

Method = Literal["linear", "logit", "probit", "poisson", "nbinom"]
VarType = Literal["delta", "simu", "bootstrap"]
VcovType = Literal["homoscedastic", "robust", "cluster", "pcse"]
TreatType = Literal["discrete", "continuous"]
XdistrType = Literal["histogram", "density", "none"]
```

### 4.2 `result.py` — InterflexResult Dataclass

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Any
import numpy as np
import pandas as pd
import matplotlib.figure

@dataclass
class InterflexResult:
    """Container for interflex linear estimator output.

    Mirrors the R S3 'interflex' class list structure.
    """
    # Core results (keyed by treatment arm label)
    est_lin: dict[str, np.ndarray]          # TE/ME table: (k, 5 or 7) [X, TE/ME, sd, lower, upper, ?uniform_lower, ?uniform_upper]
    pred_lin: dict[str, np.ndarray]         # Predicted values table: same shape
    link_lin: dict[str, np.ndarray]         # Link values table: same shape
    diff_estimate: dict[str, pd.DataFrame]  # Difference estimates table
    vcov_matrix: dict[str, np.ndarray]      # TE/ME covariance matrix (k x k) per arm
    avg_estimate: dict[str, pd.DataFrame] | pd.DataFrame  # ATE (discrete) or AME (continuous)

    # Metadata
    treat_info: dict[str, Any]
    diff_info: dict[str, Any]

    # Labels
    xlabel: str = ""
    dlabel: str = ""
    ylabel: str = ""

    # Distribution data (for plotting)
    de: Any = None          # KDE density of X
    de_tr: Any = None       # Per-treatment KDE (discrete) or None
    hist_out: Any = None    # Histogram data
    count_tr: Any = None    # Per-treatment histogram counts (discrete) or None

    # Tests and diagnostics
    tests: dict[str, Any] = field(default_factory=dict)

    # Model object
    estimator: str = "linear"
    model_linear: Any = None   # Fitted statsmodels model
    use_fe: bool = False

    # Figure
    figure: matplotlib.figure.Figure | None = None

    def predict(self, type: str = "response", **plot_kwargs) -> matplotlib.figure.Figure | None:
        """Predict method — delegates to predict_interflex()."""
        from .predict import predict_interflex
        return predict_interflex(self, type=type, **plot_kwargs)
```

### 4.3 `core.py` — Entry Point and Input Validation

```python
def interflex(
    estimator: str = "linear",
    data: pd.DataFrame = ...,
    Y: str = ...,
    D: str = ...,
    X: str = ...,
    treat_type: TreatType | None = None,
    base: str | None = None,
    Z: list[str] | None = None,
    IV: list[str] | None = None,
    FE: list[str] | None = None,
    full_moderate: bool = False,
    weights: str | None = None,
    na_rm: bool = False,
    Xunif: bool = False,
    CI: bool = True,
    neval: int = 50,
    X_eval: np.ndarray | None = None,
    method: Method = "linear",
    vartype: VarType = "delta",
    vcov_type: VcovType = "robust",
    time: str | None = None,
    pairwise: bool = True,
    nboots: int = 200,
    nsimu: int = 1000,
    parallel: bool = False,
    cores: int = 4,
    cl: str | None = None,
    Z_ref: np.ndarray | list | None = None,
    D_ref: np.ndarray | list[float] | None = None,
    diff_values: np.ndarray | None = None,
    percentile: bool = False,
    figure: bool = True,
    # ... plotting parameters omitted for brevity, see Section 4.8 ...
) -> InterflexResult:
```

**Input Validation** (translate from `interflex.R` lines 88-546):

1. Coerce `data` to DataFrame if needed
2. Validate `estimator` is `"linear"` (this package only supports linear)
3. Validate `Y`, `D`, `X` are strings present in `data.columns`
4. Validate `Z` elements are strings in `data.columns`
5. Validate `FE`: only allowed with `method="linear"`; columns must exist
6. Validate `IV`: only allowed with `method="linear"`; not compatible with `vcov_type="pcse"`
7. Validate `method` in `{"linear", "logit", "probit", "poisson", "nbinom"}`
8. Validate `vartype` in `{"delta", "simu", "bootstrap"}`
9. Validate `vcov_type` in `{"robust", "homoscedastic", "cluster", "pcse"}`
10. If `vcov_type="pcse"`: requires `cl` and `time`; not compatible with FE (switch to cluster with warning); only for `method="linear"`
11. If `vcov_type="cluster"`: requires `cl`
12. Validate `nboots`, `nsimu` are positive integers
13. Handle missing values: if `na_rm=True`, drop rows with NA in relevant columns; else raise if any NA
14. Validate `Z_ref` length matches `Z` length
15. Validate `D_ref` is numeric, length <= 9
16. Validate `diff_values` length is 2 or 3, within range of X
17. Validate outcome Y for binary methods (logit/probit: must be {0,1}), count methods (poisson/nbinom: non-negative integers)

**Preprocessing** (translate from `interflex.R` lines 548-945):

1. **Xunif**: if True, replace X with rank percentiles
2. **Factor covariates**: for factor columns in Z, create sum-contrast dummy variables (`Dummy.Covariate.1`, etc.), update Z list. Compute default Z_ref: means for numeric Z, zeros for dummy Z.
3. **Z.X interaction**: if `full_moderate=True`, create `Z_i * X` columns for each Z.
4. **Treat type detection**: if not specified, auto-detect from D: >5 unique values => "continuous", else "discrete"
5. **Discrete treatment**: encode treatment arms. Identify `base` (default: sorted first value). Create `all_treat`, `other_treat` mappings. Rename D values to `Group.1`, `Group.2`, etc.
6. **Continuous treatment**: default D_ref to median of D if not specified. Create `D_sample` dict mapping labels to values.
7. **diff_values**: default to 25th/50th/75th percentiles of X. Create `difference_name` labels.
8. **FE/cl/time factorization**: convert to integer codes (like `as.numeric(as.factor(...))`).

After preprocessing, call `interflex_linear(...)`.

### 4.4 `linear.py` — Main Linear Estimator

```python
def interflex_linear(
    data: pd.DataFrame,
    Y: str, D: str, X: str,
    treat_info: dict, diff_info: dict,
    Z: list[str] | None = None,
    weights: str | None = None,
    full_moderate: bool = False,
    Z_X: list[str] | None = None,
    FE: list[str] | None = None,
    IV: list[str] | None = None,
    neval: int = 50,
    X_eval: np.ndarray | None = None,
    method: Method = "linear",
    vartype: VarType = "simu",
    vcov_type: VcovType = "robust",
    time: str | None = None,
    pairwise: bool = True,
    nboots: int = 200,
    nsimu: int = 1000,
    parallel: bool = False,
    cores: int = 4,
    cl: str | None = None,
    Z_ref: np.ndarray | None = None,
    CI: bool = True,
    figure: bool = True,
    # plotting args...
) -> InterflexResult:
```

**Algorithm**:

#### Step 1: Evaluation Points
```
X_eval = np.linspace(data[X].min(), data[X].max(), neval)
X_eval = np.sort(np.unique(np.concatenate([X_eval, X_eval_user])))  # merge user points
neval = len(X_eval)
```

#### Step 2: Formula Construction

Build design matrix columns (not a string formula):

**Discrete treatment**:
- For each `char` in `other_treat`:
  - `D.<char>` = 1 if `data[D] == char` else 0
  - `DX.<char>` = `D.<char>` * `data[X]`
- Regressors: `[X, D.char1, DX.char1, D.char2, DX.char2, ..., Z1, Z2, ..., Z_X1, ...]`

**Continuous treatment**:
- `DX` = `data[D]` * `data[X]`
- Regressors: `[X, D, DX, Z1, Z2, ..., Z_X1, ...]`

#### Step 3: Model Fitting

**Case: no FE, no IV**:
- `method="linear"`: `statsmodels.api.GLM(y, X_design, family=Gaussian(), freq_weights=w).fit()`
- `method="logit"`: `statsmodels.api.GLM(y, X_design, family=Binomial(link=Logit()), freq_weights=w).fit()`
- `method="probit"`: `statsmodels.api.GLM(y, X_design, family=Binomial(link=Probit()), freq_weights=w).fit()`
- `method="poisson"`: `statsmodels.api.GLM(y, X_design, family=Poisson(), freq_weights=w).fit()`
- `method="nbinom"`: `statsmodels.discrete.discrete_model.NegativeBinomial(y, X_design, ...).fit()`
  - Note: statsmodels NB uses NB2 parameterization. The `loglike_method='nb2'` default matches R `MASS::glm.nb` when `theta` is also estimated.

For all GLM fits, add an intercept column (`sm.add_constant(X_design)`).

**Case: FE, no IV**:
- Call `fwl_demean(data_matrix, FE_matrix, weight)` from `fwl.py`
- Then weighted OLS on demeaned data: `np.linalg.lstsq(X_dm * sqrt_w[:, None], y_dm * sqrt_w)`
- Wrap result in a model-like object with `.params`, `.df_resid`, `.resid`

**Case: no FE, IV**:
- Build instruments: for each `iv` in IV, add `iv` and `X * iv` to the instrument set
- Use `statsmodels` or manual 2SLS:
  1. First stage: regress endogenous vars on instruments + exogenous
  2. Second stage: regress Y on predicted endogenous + exogenous

**Case: FE + IV**:
- Call `iv_fwl(Y_mat, X_endog_mat, Z_exog_mat, IV_mat, FE_mat, weight)` from `fwl.py`
- Returns coefficients and residuals

#### Step 4: Vcov Estimation

Extract `model_coef` (named coefficient array) and compute `model_vcov`:

- **No FE**:
  - `homoscedastic`: `model.cov_params()`
  - `robust`: HC1 — `model.cov_params()` with `cov_type='HC1'` or compute manually:
    $\hat{V}_{HC1} = \frac{n}{n-K}(X'X)^{-1} X' \text{diag}(\hat{e}^2) X (X'X)^{-1}$
  - `cluster`: call `vcov_cluster(model, data[cl])` from `vcov.py`
  - `pcse`: call `pcse_vcov(model, data[cl], data[time], pairwise)` from `vcov.py`

- **FE**:
  - `homoscedastic`: $\hat\sigma^2 (X_{dm}'X_{dm})^{-1}$ where $\hat\sigma^2 = \frac{e'e}{n - K - n_{FE}}$
  - `robust`: HC1 on demeaned data
  - `cluster`: cluster-robust on demeaned data (lfe-style — the vcov from `felm()` with `type="cluster"`)

Handle NA coefficients: set to 0. Handle NA vcov entries: set to 0. Symmetry check with tolerance 1e-6; if fails, fallback to homoscedastic vcov.

Expand vcov to full size of coefficient vector (if some coefficients were NA/dropped).

#### Step 5: Compute TE/ME, Predictions, Differences, ATE/AME

Call functions from `effects.py` and `variance.py` based on `vartype`.

**Simu path**: See Section 4.6
**Delta path**: See Section 4.6
**Bootstrap path**: See Section 4.6

#### Step 6: Compute Histogram and Density Data

For plotting:
- `de`: `scipy.stats.gaussian_kde(data[X], weights=w)` or unweighted
- `hist_out`: `np.histogram(data[X], bins=80)`
- For discrete: per-treatment-arm density and histogram counts
- For continuous: only overall density and histogram

#### Step 7: Assemble InterflexResult

Pack all results into `InterflexResult` dataclass. If `figure=True`, call `plot_interflex()`.

### 4.5 `effects.py` — Treatment / Marginal Effect Functions

#### `gen_general_te()`

```python
def gen_general_te(
    model_coef: dict[str, float],
    X_eval: np.ndarray,
    X: str, D: str, Z: list[str] | None, Z_ref: np.ndarray | None,
    Z_X: list[str] | None, full_moderate: bool,
    method: Method, treat_type: TreatType,
    use_fe: bool, base: str | None,
    D_sample: dict | None = None,
    # For discrete:
    char: str | None = None,
    # For continuous:
    D_ref: float | None = None,
    diff_values: np.ndarray | None = None,
    difference_name: list[str] | None = None,
) -> dict:
```

**Returns** (discrete): `{"TE": ndarray(k,), "E_pred": ndarray(k,), "E_base": ndarray(k,), "link_1": ndarray(k,), "link_0": ndarray(k,), "diff_estimate": ndarray or None}`

**Returns** (continuous): `{"ME": ndarray(k,), "E_pred": ndarray(k,), "link": ndarray(k,), "diff_estimate": ndarray or None}`

**Algorithm for no-FE discrete**:

For each $x$ in `X_eval`:
```
link_1 = coef["(Intercept)"] + x*coef[X] + coef[f"D.{char}"] + x*coef[f"DX.{char}"]
link_0 = coef["(Intercept)"] + x*coef[X]
# add Z_ref contributions
for z in Z:
    link_1 += Z_ref[z] * coef[z]
    link_0 += Z_ref[z] * coef[z]
    if full_moderate:
        link_1 += Z_ref[z] * coef[f"{z}.X"] * x
        link_0 += Z_ref[z] * coef[f"{z}.X"] * x
# apply link function
if method == "linear": TE = link_1 - link_0; E_pred = link_1; E_base = link_0
if method == "logit": E_pred = expit(link_1); E_base = expit(link_0); TE = E_pred - E_base
if method == "probit": E_pred = Phi(link_1); E_base = Phi(link_0); TE = E_pred - E_base
if method in ("poisson", "nbinom"): E_pred = exp(link_1); E_base = exp(link_0); TE = E_pred - E_base
```

Vectorize over `X_eval` using numpy broadcasting.

**Algorithm for FE discrete**: `TE = coef[f"D.{char}"] + X_eval * coef[f"DX.{char}"]`, predictions = 0.

**Algorithm for no-FE continuous**: Same structure but `link = coef["(Intercept)"] + x*coef[X] + coef[D]*D_ref + coef["DX"]*x*D_ref + ...`, then ME per method formulas.

**Algorithm for FE continuous**: `ME = coef[D] + coef["DX"] * X_eval`, predictions = 0.

**Differences**: Evaluate TE/ME at `diff_values` points and compute pairwise differences (2 points: 1 diff; 3 points: 3 diffs).

#### `gen_ate()`

```python
def gen_ate(
    data: pd.DataFrame,
    model_coef: dict[str, float],
    model_vcov: np.ndarray | None,
    X: str, D: str, Z: list[str] | None, Z_X: list[str] | None,
    method: Method, treat_type: TreatType,
    use_fe: bool, full_moderate: bool,
    char: str | None = None,
    delta: bool = False,
) -> float | dict:
```

**ATE (discrete, no FE)**: Compute TE for each observation in treatment group `char` using actual covariate values. Weighted mean.

**AME (continuous, no FE)**: Compute ME for each observation using actual covariate values. Weighted mean.

**ATE (discrete, FE)**: `TE_i = coef[f"D.{char}"] + X_i * coef[f"DX.{char}"]`, weighted mean over treatment group.

**AME (continuous, FE)**: `ME_i = coef[D] + coef["DX"] * X_i`, weighted mean.

If `delta=True`, also compute SE via delta method:
1. Compute gradient vector for each observation
2. Average gradient vectors (weighted)
3. $SE = \sqrt{\bar{g}' V \bar{g}}$

### 4.6 `variance.py` — Variance Estimation Paths

#### Simulation Variance

```python
def variance_simu(
    model_coef: dict[str, float],
    model_vcov: np.ndarray,
    coef_names: list[str],
    nsimu: int,
    X_eval: np.ndarray,
    gen_general_te_fn: Callable,
    gen_ate_fn: Callable,
    # ... all parameters needed by gen_general_te and gen_ate
) -> dict:
```

**Algorithm**:
1. `simu_coef = np.random.default_rng().multivariate_normal(coef_array, vcov_matrix, size=nsimu)` — shape (nsimu, p)
2. For each simulated coefficient vector, compute TE/ME at all evaluation points, predictions, differences, ATE/AME
3. SD = `np.std(simu_matrix, axis=1)` across simulations
4. CI = `np.percentile(simu_matrix, [2.5, 97.5], axis=1)` across simulations
5. Vcov = `np.cov(simu_matrix)` — covariance of TE/ME across evaluation points

#### Delta Method Variance

```python
def variance_delta(
    model_coef: dict[str, float],
    model_vcov: np.ndarray,
    coef_names: list[str],
    X_eval: np.ndarray,
    model_df: int,
    # ... context parameters
) -> dict:
```

**Algorithm**:
1. `crit = abs(scipy.stats.t.ppf(0.025, df=model_df))`
2. For each treatment arm and each evaluation point, compute the gradient vector `g'(x)` as specified in comprehension.md
3. `SE(x) = sqrt(g(x)' @ V @ g(x))`
4. CI = `estimate +/- crit * SE`
5. Uniform CI: call `calculate_delta_uniform_ci(TE_vcov)` to get critical value `q`, then `estimate +/- q * SE`
6. For differences: `SE_diff = sqrt((g2-g1)' @ V @ (g2-g1))`
7. For vcov across evaluation points: `cov_matrix[i,j] = g(x_i)' @ V @ g(x_j)`
8. ATE/AME with delta SE via `gen_ate(..., delta=True)`

#### Bootstrap Variance

```python
def variance_bootstrap(
    data: pd.DataFrame,
    formula_info: dict,  # Everything needed to refit the model
    nboots: int,
    cl: str | None,
    X_eval: np.ndarray,
    gen_general_te_fn: Callable,
    gen_ate_fn: Callable,
    parallel: bool,
    cores: int,
    # ... context parameters
) -> dict:
```

**Algorithm**:
1. For each bootstrap replicate b = 1..nboots:
   a. If no clustering: sample n indices with replacement
   b. If clustering: sample cluster IDs with replacement, take all observations from selected clusters
   c. Check bootstrap sample validity (discrete: all treatment arms present; GLM: convergence)
   d. Refit model on bootstrap sample
   e. Compute TE/ME, predictions, differences, ATE/AME using bootstrap coefficients
2. Across replicates: SD, percentile CI, cov matrix
3. Uniform CI: call `calculate_uniform_quantiles(boot_matrix, 0.05)`
4. Parallel: use `concurrent.futures.ProcessPoolExecutor` if `parallel=True`

### 4.7 `vcov.py` — Variance-Covariance Estimators

#### `vcov_cluster()`

```python
def vcov_cluster(
    X: np.ndarray,          # Design matrix (n, p) — from model
    residuals: np.ndarray,  # (n,)
    cluster: np.ndarray,    # (n,) cluster labels
    coef_names: list[str],  # coefficient names for alignment
    rank: int,              # model rank K
) -> np.ndarray:
```

**Algorithm** (direct translation of `vcluster.R`):

1. `M` = number of unique clusters
2. `N` = number of observations
3. `K` = model rank
4. `dfc = (M / (M-1)) * ((N-1) / (N-K))`
5. Compute score matrix: `S_i = X_i * e_i` (element-wise, each row is one observation's score contribution)
6. Aggregate scores by cluster: `u_c = sum(S_i for i in cluster c)` — shape (M, p)
7. Meat: `meat = u_c.T @ u_c / N`
8. Bread: `bread = inv(X.T @ X / N)` (or use model's bread matrix)
9. `V_cluster = dfc * bread @ meat @ bread`

#### `robust_vcov()`

```python
def robust_vcov(
    X: np.ndarray,
    residuals: np.ndarray,
    rank: int,
) -> np.ndarray:
```

HC1: $\hat{V}_{HC1} = \frac{n}{n-K} (X'X)^{-1} X' \text{diag}(\hat{e}^2) X (X'X)^{-1}$

#### `pcse_vcov()`

```python
def pcse_vcov(
    model_residuals: np.ndarray,
    X: np.ndarray,
    group_n: np.ndarray,  # panel unit IDs
    group_t: np.ndarray,  # time period IDs
    pairwise: bool = True,
) -> np.ndarray:
```

Panel-corrected standard errors (Beck & Katz 1995). Implementation follows the `pcse` R package logic:
1. Organize residuals into a (T x N) panel matrix
2. Estimate cross-unit error covariance: $\hat\Omega_{ij} = \frac{1}{T_{ij}} \sum_t e_{it} e_{jt}$, where $T_{ij}$ is the number of common time periods (pairwise) or total T (complete)
3. Construct block-diagonal sandwich: $\hat{V}_{PCSE} = (X'X)^{-1} X' (\hat\Omega \otimes I_T) X (X'X)^{-1}$

### 4.8 `fwl.py` — Frisch-Waugh-Lovell Demeaning

#### `fwl_demean()`

```python
def fwl_demean(
    data: np.ndarray,       # (n, k) — Y in col 0, covariates in cols 1..k-1
    FE: np.ndarray,         # (n, m) — FE group indicators (integer-coded)
    weight: np.ndarray,     # (n,) — observation weights
    tol: float = 1e-5,
    max_iter: int = 50,
) -> tuple[np.ndarray, np.ndarray, int, float]:
    """
    Returns:
        coefficients: (p,) — OLS coefficients on demeaned data
        residuals: (n,) — residuals (unweighted scale)
        niter: number of iterations
        mu: grand mean
    """
```

**Algorithm** (exact translation of `fastplm.cpp`):

```
data_old = zeros(n, k)
diff = 100.0
niter = 0

while diff > tol and niter < max_iter:
    for each FE column m:
        # weighted data for this iteration
        data_wei = data * weight[:, None]

        # compute weighted group means
        for each unique group g in FE[:, m]:
            mask = (FE[:, m] == g)
            sum_w = weight[mask].sum()
            if sum_w > 0:
                group_mean = (data_wei[mask]).sum(axis=0) / sum_w
            else:
                group_mean = 0
            # subtract group mean from all observations in this group
            data[mask] -= group_mean

        data_wei = data  # update for next FE

    diff = np.abs(data - data_old).sum()
    data_old = data.copy()
    niter += 1

# Apply sqrt(weight) for WLS
y = data[:, 0] * np.sqrt(weight)
X = data[:, 1:] * np.sqrt(weight)[:, None]

# Drop zero-variance columns (set coef to NaN)
# OLS: coef = np.linalg.lstsq(X, y, rcond=None)[0]
# residuals = (data[:, 0] * sqrt(w) - X @ coef) / sqrt(w)

# Grand mean
mu = (data_bak[:, 0] * weight).sum() / weight.sum()
if p > 0:
    X_wmean = (data_bak[:, 1:] * weight[:, None]).sum(axis=0) / weight.sum()
    mu -= X_wmean @ coef

return coefficients, residuals, niter, mu
```

#### `iv_fwl()`

```python
def iv_fwl(
    Y: np.ndarray,     # (n, 1) — outcome
    X_endog: np.ndarray,  # (n, k_X) — endogenous regressors
    Z_exog: np.ndarray,   # (n, k_Z) — exogenous regressors
    IV: np.ndarray,       # (n, k_IV) — instruments
    FE: np.ndarray,       # (n, m) — FE indicators
    weight: np.ndarray,   # (n,)
    tol: float = 1e-5,
    max_iter: int = 50,
) -> tuple[np.ndarray, np.ndarray, int, float]:
```

**Algorithm** (exact translation of `iv_fastplm.cpp`):

1. Concatenate `[Y, X_endog, Z_exog, IV]` into data matrix
2. Apply same iterative FWL demeaning as `fwl_demean()`
3. Apply `sqrt(weight)` to demeaned data
4. Recover demeaned Y, X_endog, Z_exog, IV
5. `Z_full = [Z_exog, IV]`
6. `X_full = [X_endog, Z_exog]`
7. `inv_ZZ = np.linalg.inv(Z_full.T @ Z_full)`
8. `PzX = Z_full @ inv_ZZ @ (Z_full.T @ X_full)`
9. `PzY = Z_full @ inv_ZZ @ (Z_full.T @ Y)`
10. `coef = np.linalg.lstsq(PzX, PzY, rcond=None)[0]`
11. `residuals = (Y - X_full @ coef) / sqrt(weight)`
12. Grand mean same as OLS case

### 4.9 `uniform.py` — Uniform Confidence Intervals

#### `calculate_uniform_quantiles()`

```python
def calculate_uniform_quantiles(
    theta_matrix: np.ndarray,  # (k, N) — k evaluation points, N bootstrap/simulation draws
    alpha: float = 0.05,
) -> tuple[np.ndarray, float, float]:
    """
    Returns:
        Q_j: (k, 2) — lower and upper uniform quantile bounds per evaluation point
        zeta_hat: the calibrated zeta
        coverage: actual coverage fraction
    """
```

**Algorithm**:
1. Drop columns with any NaN
2. k = number of rows (evaluation points), N = number of columns (draws)
3. Binary search for `zeta` in `[alpha/(2*k), alpha/2]`:
   - For candidate zeta: compute pointwise (zeta, 1-zeta) quantiles per row
   - Count fraction of draws where ALL rows fall within their bands
   - If coverage < 1-alpha: decrease zeta (tighten search upper bound)
   - Else: increase zeta (raise search lower bound)
   - Tolerance: 1e-6
4. If zeta_hat equals Bonferroni bound alpha/(2*k), warn
5. Return quantile bands at final zeta_hat

#### `calculate_delta_uniform_ci()`

```python
def calculate_delta_uniform_ci(
    Sigma_hat: np.ndarray,  # (k, k) — TE/ME covariance matrix
    alpha: float = 0.05,
    N: int = 2000,
) -> float:
    """Returns the critical value q for uniform CI."""
```

**Algorithm**:
1. k = Sigma_hat.shape[0]
2. `V = np.random.default_rng().multivariate_normal(np.zeros(k), Sigma_hat, size=N)` — shape (N, k)
3. `Sigma_inv_sqrt = 1.0 / np.sqrt(np.diag(Sigma_hat))`
4. `max_vals = np.max(np.abs(V * Sigma_inv_sqrt), axis=1)` — shape (N,)
5. `q = np.percentile(max_vals, 100*(1-alpha))`
6. Return q

### 4.10 `plotting.py` — Plot Functions

```python
def plot_interflex(
    result: InterflexResult,
    order: list[str] | None = None,
    subtitles: list[str] | None = None,
    show_subtitles: bool | None = None,
    CI: bool | None = None,
    diff_values: np.ndarray | None = None,
    Xdistr: XdistrType = "histogram",
    main: str | None = None,
    Ylabel: str | None = None,
    Dlabel: str | None = None,
    Xlabel: str | None = None,
    xlab: str | None = None,
    ylab: str | None = None,
    xlim: tuple[float, float] | None = None,
    ylim: tuple[float, float] | None = None,
    theme_bw: bool = False,
    show_grid: bool = True,
    cex_main: float | None = None,
    cex_sub: float | None = None,
    cex_lab: float | None = None,
    cex_axis: float | None = None,
    interval: np.ndarray | None = None,
    file: str | None = None,
    ncols: int | None = None,
    pool: bool = False,
    color: list[str] | None = None,
    show_all: bool = False,
    scale: float = 1.1,
    height: float = 7.0,
    width: float = 10.0,
) -> matplotlib.figure.Figure:
```

**Layout**:
- One subplot per treatment arm (discrete) or per D_ref value (continuous)
- `ncols` controls grid layout
- If `pool=True`, all treatment arms on one plot with legend

**Each subplot**:
1. Line plot: TE/ME or E(Y) vs X
2. Shaded ribbon: pointwise 95% CI (alpha=0.3)
3. Dashed ribbon: uniform 95% CI (if available)
4. Bottom strip: histogram bars or density curve of X distribution (scaled to bottom 1/5 of y range)
5. Optional: vertical lines at `diff_values`

**Colors**: Use `seaborn` Dark2-like palette. Override with `color` parameter.

**Saving**: If `file` is provided, save to file with `fig.savefig(file, dpi=100, bbox_inches='tight')`.

### 4.11 `predict.py` — Predict Method

```python
def predict_interflex(
    result: InterflexResult,
    type: str = "response",  # "response" or "link"
    order: list[str] | None = None,
    subtitles: list[str] | None = None,
    show_subtitles: bool | None = None,
    CI: bool | None = None,
    pool: bool = False,
    # ... same plotting kwargs as plot_interflex
) -> matplotlib.figure.Figure | None:
```

**Algorithm**:
1. If `result.use_fe`: return None (cannot predict levels with FE)
2. Extract `pred_lin` (type="response") or `link_lin` (type="link") from result
3. Call plot function with the extracted prediction data

## 5. Numerical Constraints

1. **FWL convergence**: tolerance 1e-5, max 50 iterations — match C++ exactly
2. **Vcov symmetry check**: tolerance 1e-6. If asymmetric, fall back to homoscedastic vcov.
3. **NaN handling**: set NaN coefficients to 0, NaN vcov entries to 0 (matches R behavior)
4. **Delta method CI**: use t-distribution critical value `scipy.stats.t.ppf(0.025, df=model_df)`, not normal (matches R `qt(0.025, df)`)
5. **Simulation/bootstrap CI**: use empirical 2.5th/97.5th percentiles
6. **Uniform CI bisection**: tolerance 1e-6 on zeta, search in `[alpha/(2k), alpha/2]`
7. **Uniform CI delta**: 2000 MVN draws by default
8. **Bootstrap**: default 200 replicates for bootstrap, 1000 for simulation
9. **Weight handling**: weights default to all-ones. Apply via `freq_weights` in statsmodels GLM, via manual WLS for FWL.
10. **Zero-variance columns**: after FWL demeaning, drop columns with zero variance (set coefficient to NaN). Match C++ behavior where `arma::unique(X.col(i)).n_rows == 1`.

## 6. API Surface

### Public API (`__init__.py`)

```python
from .core import interflex
from .result import InterflexResult

__all__ = ["interflex", "InterflexResult"]
__version__ = "0.1.0"
```

### Function Signatures Summary

| Function | Module | Inputs | Output |
|----------|--------|--------|--------|
| `interflex()` | `core.py` | DataFrame + all parameters | `InterflexResult` |
| `interflex_linear()` | `linear.py` | Preprocessed data + params | `InterflexResult` |
| `gen_general_te()` | `effects.py` | coefs, X_eval, context | dict of arrays |
| `gen_ate()` | `effects.py` | data, coefs, vcov, context | float or dict |
| `gen_delta_te()` | `variance.py` | coefs, vcov, X_eval, context | dict of arrays |
| `variance_simu()` | `variance.py` | coefs, vcov, nsimu, context | dict |
| `variance_bootstrap()` | `variance.py` | data, nboots, context | dict |
| `vcov_cluster()` | `vcov.py` | X, residuals, cluster, rank | ndarray |
| `robust_vcov()` | `vcov.py` | X, residuals, rank | ndarray |
| `pcse_vcov()` | `vcov.py` | residuals, X, panel IDs, time | ndarray |
| `fwl_demean()` | `fwl.py` | data, FE, weight | (coefs, resid, niter, mu) |
| `iv_fwl()` | `fwl.py` | Y, X, Z, IV, FE, weight | (coefs, resid, niter, mu) |
| `calculate_uniform_quantiles()` | `uniform.py` | matrix, alpha | (Q_j, zeta, coverage) |
| `calculate_delta_uniform_ci()` | `uniform.py` | Sigma, alpha, N | float |
| `plot_interflex()` | `plotting.py` | InterflexResult, kwargs | Figure |
| `predict_interflex()` | `predict.py` | InterflexResult, type, kwargs | Figure or None |

## 7. Implementation Notes

### R-to-Python Gotchas

1. **R indexing is 1-based**. All array references in the R code must be adjusted.
2. **R's `glm()` returns named coefficients** including `"(Intercept)"`. In Python, use `sm.add_constant()` and maintain a name mapping.
3. **R's `felm()` from `lfe` package** uses Rcpp under the hood. Our `fwl_demean()` replaces it.
4. **R's `ivreg()` from `AER` package**. Our `iv_fwl()` or manual 2SLS replaces it.
5. **R's `MASS::glm.nb()`** estimates theta internally. `statsmodels.NegativeBinomial` also estimates the dispersion but with a different parameterization (NB2 with `alpha` = 1/theta). The builder must ensure `alpha = 1/theta` mapping.
6. **R's `sandwich::vcovHC(type="HC1")`** uses the standard HC1 formula. statsmodels `HC1` matches.
7. **R's `mvtnorm::rmvnorm()`** draws from multivariate normal. Use `np.random.default_rng().multivariate_normal()`.
8. **R's `dnorm()`/`pnorm()`** are standard normal pdf/cdf. Use `scipy.stats.norm.pdf()`/`.cdf()`.
9. **R's `qt()`** is the t-distribution quantile. Use `scipy.stats.t.ppf()`.
10. **R's named vectors/lists**: Use Python dicts with string keys.
11. **R formula `Y ~ X + D.char + DX.char | FE1 + FE2`**: In Python, construct design matrices manually.
12. **R's `weighted.mean()`**: `np.average(values, weights=weights)`.
13. **R's `Lmoments::Lmoments()`**: For L-kurtosis, use `scipy.stats` or a direct computation. This is diagnostic only and does not affect numerical results.

### Performance Considerations

- FWL demeaning: use vectorized numpy operations, not Python loops over observations. Group-by operations via `np.bincount` or `pandas.groupby`.
- Bootstrap: use `concurrent.futures.ProcessPoolExecutor` for parallel bootstrap.
- Simulation variance: fully vectorized — `simu_coef` matrix times design vectors.

### Coefficient Naming Convention

Maintain a consistent naming scheme that mirrors R:
- `"(Intercept)"` for the intercept (no-FE models only)
- `X` (the actual column name) for the moderator coefficient
- `f"D.{char}"` for discrete treatment dummies
- `f"DX.{char}"` for discrete treatment x moderator interactions
- `D` (actual column name) for continuous treatment
- `"DX"` for continuous treatment x moderator interaction
- `z_name` for each covariate
- `f"{z_name}.X"` for covariate x moderator interactions (full_moderate)
