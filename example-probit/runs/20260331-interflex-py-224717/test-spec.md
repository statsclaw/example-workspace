# Test Pipeline Specification — interflex Python Package

## 1. Behavioral Contract

The `interflex` Python package must:

1. Accept a pandas DataFrame with outcome (Y), treatment (D), and moderator (X) columns, plus optional covariates, cluster variable, and weights.
2. Detect whether treatment is discrete or continuous (or accept explicit specification).
3. Fit a linear interaction model via OLS/WLS.
4. Compute treatment effects (discrete) or marginal effects (continuous) at a grid of moderator values.
5. Compute standard errors via delta method, simulation, or bootstrap.
6. Compute average treatment effects (ATE) or average marginal effects (AME).
7. Compute difference estimates at specified moderator values.
8. Support homoscedastic, HC1 robust, and cluster-robust variance estimation.
9. Return results in a structured dict with pandas DataFrames.
10. Raise clear errors for invalid inputs.

---

## 2. Test Scenarios

### 2.1 Shared Test Data Fixtures

All fixtures should be defined in `conftest.py` with deterministic data generation using fixed seeds.

#### Fixture A: Simple Discrete Treatment (Binary)

```python
np.random.seed(42)
n = 200
X = np.random.uniform(0, 10, n)
D = np.random.choice(["control", "treated"], n)
Z1 = np.random.normal(0, 1, n)
epsilon = np.random.normal(0, 1, n)
# True DGP: Y = 2 + 0.5*X + 3*D_treated + 0.8*D_treated*X + 0.4*Z1 + epsilon
D_num = (D == "treated").astype(float)
Y = 2.0 + 0.5*X + 3.0*D_num + 0.8*D_num*X + 0.4*Z1 + epsilon
data = pd.DataFrame({"Y": Y, "D": D, "X": X, "Z1": Z1})
```

True parameters:
- intercept = 2.0
- beta_X = 0.5
- beta_D.treated = 3.0
- beta_DX.treated = 0.8
- beta_Z1 = 0.4

True TE(x) = 3.0 + 0.8*x
True ATE (over treated group) = 3.0 + 0.8 * mean(X[D=="treated"])

#### Fixture B: Continuous Treatment

```python
np.random.seed(123)
n = 300
X = np.random.uniform(-5, 5, n)
D = np.random.normal(2, 1, n)
epsilon = np.random.normal(0, 0.5, n)
# True DGP: Y = 1 + 0.3*X + 1.5*D + 0.4*D*X + epsilon
Y = 1.0 + 0.3*X + 1.5*D + 0.4*D*X + epsilon
data = pd.DataFrame({"Y": Y, "D": D, "X": X})
```

True ME(x) = 1.5 + 0.4*x
True AME = 1.5 + 0.4 * mean(X)

#### Fixture C: Multi-valued Discrete Treatment (3 groups)

```python
np.random.seed(77)
n = 300
X = np.random.uniform(0, 5, n)
D = np.random.choice(["A", "B", "C"], n, p=[0.4, 0.3, 0.3])
epsilon = np.random.normal(0, 0.8, n)
# True DGP: base="A"
# Y = 1 + X + 2*D_B + 0.5*D_B*X + 4*D_C + 1.0*D_C*X + epsilon
D_B = (D == "B").astype(float)
D_C = (D == "C").astype(float)
Y = 1.0 + 1.0*X + 2.0*D_B + 0.5*D_B*X + 4.0*D_C + 1.0*D_C*X + epsilon
data = pd.DataFrame({"Y": Y, "D": D, "X": X})
```

True TE_B(x) = 2.0 + 0.5*x
True TE_C(x) = 4.0 + 1.0*x

#### Fixture D: Clustered Data

```python
np.random.seed(99)
n_clusters = 50
obs_per_cluster = 10
n = n_clusters * obs_per_cluster
cluster_id = np.repeat(np.arange(n_clusters), obs_per_cluster)
cluster_effect = np.repeat(np.random.normal(0, 2, n_clusters), obs_per_cluster)
X = np.random.uniform(0, 10, n)
D = np.random.choice([0, 1], n)
epsilon = np.random.normal(0, 1, n)
Y = 1.0 + 0.5*X + 2.0*D + 0.3*D*X + cluster_effect + epsilon
data = pd.DataFrame({"Y": Y, "D": D.astype(str), "X": X, "cl": cluster_id})
```

#### Fixture E: Weighted Data

```python
np.random.seed(55)
n = 200
X = np.random.uniform(0, 10, n)
D = np.random.choice(["ctrl", "trt"], n)
w = np.random.uniform(0.5, 2.0, n)
epsilon = np.random.normal(0, 1, n)
D_num = (D == "trt").astype(float)
Y = 1.0 + 0.5*X + 2.5*D_num + 0.6*D_num*X + epsilon
data = pd.DataFrame({"Y": Y, "D": D, "X": X, "weights": w})
```

#### Fixture F: With Covariates and Full Moderation

```python
np.random.seed(33)
n = 250
X = np.random.uniform(0, 8, n)
D = np.random.choice(["low", "high"], n)
Z1 = np.random.normal(0, 1, n)
Z2 = np.random.normal(5, 2, n)
epsilon = np.random.normal(0, 1, n)
D_num = (D == "high").astype(float)
# full moderate: Y = a + bX + cD + dDX + e1*Z1 + e2*Z2 + f1*Z1*X + f2*Z2*X + eps
Y = (1.0 + 0.5*X + 3.0*D_num + 0.7*D_num*X
     + 0.4*Z1 + 0.2*Z2
     + 0.1*Z1*X + 0.05*Z2*X
     + epsilon)
data = pd.DataFrame({"Y": Y, "D": D, "X": X, "Z1": Z1, "Z2": Z2})
```

---

### 2.2 Core Functionality Tests (`test_basic.py`)

#### Test 2.2.1: Binary Discrete Treatment — Point Estimates

Using Fixture A, call `interflex(estimator="linear", data=data, Y="Y", D="D", X="X", Z=["Z1"], vartype="delta", vcov_type="robust")`.

Verify:
- Result is a dict with keys: `"treat_info"`, `"est_lin"`, `"pred_lin"`, `"diff_estimate"`, `"vcov_matrix"`, `"avg_estimate"`, `"model"`.
- `result["treat_info"]["treat_type"] == "discrete"`.
- `result["est_lin"]` has exactly one key (the treatment label, "treated" or equivalent).
- The est_lin DataFrame has columns including X, TE/ME, sd.
- TE values at X=0 should be approximately 3.0 (within +/- 1.0, given noise).
- TE values at X=5 should be approximately 3.0 + 0.8*5 = 7.0 (within +/- 1.5).
- The TE should be monotonically increasing with X (since beta_DX > 0).

Tolerance: point estimates within 1.5 of true values (large tolerance because of noise in n=200 sample).

#### Test 2.2.2: Continuous Treatment — Point Estimates

Using Fixture B, call `interflex(estimator="linear", data=data, Y="Y", D="D", X="X", vartype="delta")`.

Verify:
- `result["treat_info"]["treat_type"] == "continuous"` (auto-detected because D has >5 unique values).
- ME values at X=0 should be approximately 1.5 (within +/- 0.5).
- ME values at X=2 should be approximately 1.5 + 0.4*2 = 2.3 (within +/- 0.5).

#### Test 2.2.3: Multi-Group Discrete Treatment

Using Fixture C, call `interflex(estimator="linear", data=data, Y="Y", D="D", X="X", base="A")`.

Verify:
- `result["est_lin"]` has exactly 2 keys (groups B and C, since A is base).
- TE for group B at X=0 is approximately 2.0.
- TE for group C at X=0 is approximately 4.0.
- TE for group C at X=3 is approximately 4.0 + 1.0*3 = 7.0.
- Base group predictions are in `result["pred_lin"]` under the "A" key.

Tolerance: within 1.5 of true values.

#### Test 2.2.4: Treatment Type Auto-Detection

- Discrete: Create data where D has values ["a", "b"]. Verify `treat_type == "discrete"`.
- Discrete: Create data where D has numeric values [0, 1, 2]. Verify `treat_type == "discrete"` (<=5 unique values).
- Continuous: Create data where D has 100 unique numeric values. Verify `treat_type == "continuous"`.

#### Test 2.2.5: Base Group Selection

Using Fixture C:
- Without `base`: base should be "A" (first sorted value).
- With `base="B"`: base should be "B", est_lin should have keys for "A" and "C".
- With `base="invalid"`: should raise ValueError.

#### Test 2.2.6: Default Diff Values

Using Fixture A, call without `diff_values`:
- `result["diff_estimate"]` should exist.
- It should have 3 rows with names containing "vs" (the 25/50/75 quantile comparisons).
- diff_estimate values should be numeric and non-NaN.
- z-value and p-value columns should exist and be numeric.

#### Test 2.2.7: Custom Diff Values (2 values)

Using Fixture A, call with `diff_values=[2.0, 8.0]`:
- `result["diff_estimate"]` should have 1 row.
- diff_estimate should be approximately TE(8) - TE(2) = 0.8*(8-2) = 4.8 (within +/- 1.5).

#### Test 2.2.8: Custom Diff Values (3 values)

Using Fixture A, call with `diff_values=[1.0, 5.0, 9.0]`:
- `result["diff_estimate"]` should have 3 rows.
- Names should include "5.0 vs 1.0", "9.0 vs 5.0", "9.0 vs 1.0".

#### Test 2.2.9: Weighted Estimation

Using Fixture E, call with `weights="weights"`:
- Result should be a valid output dict.
- Point estimates should be close to true values (within 2.0).
- Weighted estimates may differ slightly from unweighted.

#### Test 2.2.10: Xunif Transform

Using Fixture A, call with `Xunif=True`:
- The X values in the output should range from approximately 0 to 100 (percentile scale).
- The shape of TE(x) should still be linear.

---

### 2.3 Variance Estimation Tests (`test_variance.py`)

#### Test 2.3.1: Delta Method — SE Sign and Magnitude

Using Fixture A with `vartype="delta"`:
- All sd values in est_lin should be positive.
- All sd values should be less than 5.0 (reasonable upper bound for this DGP).
- CI should satisfy: lower_CI < TE < upper_CI for all evaluation points.

#### Test 2.3.2: Simulation Variance — SE Sign and Magnitude

Using Fixture A with `vartype="simu"`, `nsimu=2000`:
- All sd values should be positive.
- sd values should be comparable to delta method SEs (within factor of 2).
- CI should be based on quantiles (may not be symmetric around point estimate).

#### Test 2.3.3: Bootstrap Variance — SE Sign and Magnitude

Using Fixture A with `vartype="bootstrap"`, `nboots=500`:
- All sd values should be positive.
- sd values should be comparable to delta method SEs (within factor of 3).
- CI should cover the true TE at most evaluation points (at least 80% coverage for a large enough sample).

#### Test 2.3.4: Delta vs Simulation Agreement (Large Sample)

Generate a large sample (n=5000) from Fixture A's DGP. Compute SEs with both `vartype="delta"` and `vartype="simu"` (nsimu=5000).

Verify: For each evaluation point, `|delta_sd - simu_sd| / delta_sd < 0.15` (within 15% relative).

This tests that simulation-based variance converges to the delta method for linear models.

#### Test 2.3.5: Homoscedastic Vcov

Using Fixture A with `vcov_type="homoscedastic"`:
- SEs should be produced without error.
- Homoscedastic SEs may differ from robust SEs.

#### Test 2.3.6: HC1 Robust Vcov

Using Fixture A with `vcov_type="robust"`:
- SEs should be produced without error.
- For this DGP (homoscedastic errors), robust SEs should be similar to homoscedastic SEs (within factor of 1.5).

#### Test 2.3.7: Cluster-Robust Vcov

Using Fixture D with `vcov_type="cluster"`, `cl="cl"`:
- SEs should be produced without error.
- Cluster-robust SEs should generally be larger than non-clustered SEs (due to intra-cluster correlation).
- Verify by comparing: run with `vcov_type="robust"` and `vcov_type="cluster"` — cluster SEs should be larger on average.

#### Test 2.3.8: Cluster Bootstrap (Block Bootstrap)

Using Fixture D with `vartype="bootstrap"`, `cl="cl"`, `nboots=200`:
- Bootstrap should resample clusters, not individual observations.
- SEs should be positive and finite.
- SEs should be comparable to cluster-robust delta method SEs (within factor of 3).

#### Test 2.3.9: Vcov Matrix Output

Using Fixture A with `vartype="delta"`:
- `result["vcov_matrix"]` should contain a square matrix of size (neval, neval).
- The matrix should be symmetric.
- Diagonal entries should be positive (variances).
- Diagonal entries should equal sd^2 (within floating-point tolerance 1e-8).

#### Test 2.3.10: CI Coverage Property

Generate 100 datasets from Fixture A's DGP (different seeds). For each dataset, compute 95% CI at X=5.0 using delta method. Count how many CIs contain the true TE(5) = 3.0 + 0.8*5 = 7.0.

Verify: coverage rate is between 0.85 and 1.00 (95% CI should cover the truth approximately 95% of the time; with n=200, we allow some slack).

---

### 2.4 Treatment Effect Computation Tests (`test_effects.py`)

#### Test 2.4.1: TE Linearity (Discrete)

Using Fixture A, verify that TE(x) is a linear function of x:
- Extract TE values at X_eval points.
- Fit a line to (X_eval, TE) using `np.polyfit(X_eval, TE, 1)`.
- Verify R^2 > 0.99 (TE is linear in X).

#### Test 2.4.2: ME Linearity (Continuous)

Using Fixture B, verify that ME(x) is a linear function of x:
- Extract ME values at X_eval points.
- Fit a line to (X_eval, ME) using `np.polyfit(X_eval, ME, 1)`.
- Verify R^2 > 0.99.

#### Test 2.4.3: TE at Extreme X Values

Using Fixture A, check that TE at min(X) and max(X) are consistent with the linear relationship.

#### Test 2.4.4: Predicted Values (Discrete)

Using Fixture A:
- `result["pred_lin"]` should have entries for both treatment groups (base and treated).
- E(Y|treated, X=x) should be approximately 2 + 0.5*x + 3 + 0.8*x + 0.4*mean(Z1) at X=x (within noise tolerance).
- E(Y|control, X=x) should be approximately 2 + 0.5*x + 0.4*mean(Z1) at X=x.
- The difference E(Y|treated) - E(Y|control) should equal TE.

#### Test 2.4.5: ATE/AME Values

Using Fixture A:
- ATE should be approximately 3.0 + 0.8 * mean(X[treated]) (within 1.5).

Using Fixture B:
- AME should be approximately 1.5 + 0.4 * mean(X) (within 0.5).

#### Test 2.4.6: ATE with Delta-Method SE

Using Fixture A with `vartype="delta"`:
- `result["avg_estimate"]` should contain ATE, sd, z-value, p-value, lower_CI, upper_CI.
- sd should be positive and less than 2.0.
- z-value should be ATE/sd.
- p-value should be 2 * (1 - norm.cdf(|z|)).
- CI should contain ATE.

#### Test 2.4.7: Diff Estimate Consistency

Using Fixture A with `diff_values=[2.0, 5.0, 8.0]`:
- diff[0] = TE(5) - TE(2) should be approximately 0.8*3 = 2.4.
- diff[1] = TE(8) - TE(5) should be approximately 0.8*3 = 2.4.
- diff[2] = TE(8) - TE(2) should be approximately 0.8*6 = 4.8.
- diff[2] should equal diff[0] + diff[1] (additivity).

Tolerance: within 0.3 of expected values.

#### Test 2.4.8: Z_ref Effect on Predictions

Using Fixture A:
- Call with `Z_ref=[0.0]` and again with `Z_ref=[2.0]`.
- TEs should be the same (Z cancels in linear TE).
- Predicted values should differ by approximately 0.4 * 2.0 = 0.8.

#### Test 2.4.9: Full Moderation

Using Fixture F with `full_moderate=True`, `Z=["Z1", "Z2"]`:
- The result should include predictions.
- The model should have additional Z*X interaction terms.
- Estimated coefficients should be close to the true values (within 1.0).

#### Test 2.4.10: Continuous D_ref

Using Fixture B with `D_ref=[1.0, 2.0, 3.0]`:
- `result["est_lin"]` should have 3 entries (one per D_ref value).
- ME should be the same across D_ref values (ME does not depend on D_ref for linear method).
- Predicted values should differ across D_ref values.

#### Test 2.4.11: neval Parameter

Using Fixture A:
- Call with `neval=10`: est_lin should have 10 rows.
- Call with `neval=100`: est_lin should have 100 rows.
- The X values should span from min(X) to max(X).

#### Test 2.4.12: Custom X_eval

Using Fixture A with `X_eval=np.array([1.0, 5.0, 9.0])`, `neval=10`:
- The output X values should include 1.0, 5.0, and 9.0 (merged with the linspace grid).
- Total number of points should be >= 10 (linspace) + 3 (user points), minus duplicates.

---

### 2.5 Edge Case Scenarios

#### Test 2.5.1: Missing Values — Error Without na_rm

Create data with one NaN in Y. Call `interflex()` with `na_rm=False`.
- Should raise ValueError mentioning missing values.

#### Test 2.5.2: Missing Values — Removed With na_rm

Create data with one NaN in Y. Call with `na_rm=True`.
- Should succeed with n-1 observations.
- Result should be valid.

#### Test 2.5.3: Invalid Column Names

- `Y="nonexistent"` should raise ValueError.
- `D="nonexistent"` should raise ValueError.
- `X="nonexistent"` should raise ValueError.
- `Z=["nonexistent"]` should raise ValueError.

#### Test 2.5.4: Invalid Estimator

- `estimator="kernel"` should raise ValueError.
- `estimator="binning"` should raise ValueError.

#### Test 2.5.5: Invalid Vartype

- `vartype="unknown"` should raise ValueError.

#### Test 2.5.6: Invalid Vcov Type

- `vcov_type="pcse"` should raise ValueError (not supported).
- `vcov_type="unknown"` should raise ValueError.

#### Test 2.5.7: Cluster Without cl

- `vcov_type="cluster"` with `cl=None` should raise ValueError.

#### Test 2.5.8: Too Many Treatment Groups

Create data with D having 10 unique string values. Should raise ValueError (max 9 groups).

#### Test 2.5.9: Single Treatment Group

Create data where all D values are the same. The model should either raise a sensible error or return degenerate results.

#### Test 2.5.10: Very Small Sample

Create data with n=5 observations. The model should either fit (possibly with large SEs) or raise a sensible error about insufficient data.

---

### 2.6 Property-Based Invariants

#### Test 2.6.1: TE Symmetry Under Relabeling

For a binary treatment, swap the labels of "control" and "treated" and change the base group accordingly. The TEs should be negated (TE_new = -TE_old at each x).

#### Test 2.6.2: ME Independence from D_ref

For continuous treatment with linear method, ME(x) = beta_D + beta_DX * x. This is independent of D_ref. Verify that ME values are identical across different D_ref values.

#### Test 2.6.3: Weight = 1 Equivalence

Run with `weights=None` and with a weights column of all 1.0. Results should be numerically identical (within 1e-10).

#### Test 2.6.4: Additive Diff Property

For diff_values [x1, x2, x3]:
- diff(x3, x1) = diff(x2, x1) + diff(x3, x2)

This must hold exactly (within 1e-10) since all are linear functions of the same coefficients.

#### Test 2.6.5: Vcov Matrix Positive Semi-Definiteness

For any result with vcov_matrix output:
- All eigenvalues should be >= -1e-10 (allow small numerical error).

#### Test 2.6.6: Prediction Consistency

For discrete treatment: E(Y|treated, X=x) - E(Y|control, X=x) should equal TE(x) at every evaluation point.

Tolerance: 1e-10 (exact up to floating point).

---

### 2.7 Cross-Reference Benchmarks

#### Test 2.7.1: R Package Numerical Comparison (Manual)

Precompute results from R's interflex package for Fixture A parameters and store as expected values.

Using the R package:
```r
library(interflex)
set.seed(42)
n <- 200
X <- runif(n, 0, 10)
D <- sample(c("control", "treated"), n, replace=TRUE)
Z1 <- rnorm(n, 0, 1)
D_num <- as.numeric(D == "treated")
Y <- 2 + 0.5*X + 3*D_num + 0.8*D_num*X + 0.4*Z1 + rnorm(n, 0, 1)
data <- data.frame(Y=Y, D=D, X=X, Z1=Z1)
result <- interflex("linear", data=data, Y="Y", D="D", X="X", Z="Z1",
                     vartype="delta", vcov.type="robust", figure=FALSE)
# Extract est.lin, diff.estimate, Avg.estimate
```

Note: The R and Python RNG differ, so the same seed will produce different random data. For cross-validation, use a FIXED dataset (save to CSV from R, load in Python) rather than relying on seeds.

**Implementation approach**: Create a small fixed dataset (10 rows) in both tests. This dataset will be hardcoded so no RNG differences matter.

```python
# Hardcoded cross-reference dataset
data_xref = pd.DataFrame({
    "Y": [3.1, 5.2, 7.8, 2.4, 6.1, 4.3, 8.9, 1.2, 5.5, 7.0],
    "D": ["ctrl", "trt", "trt", "ctrl", "trt", "ctrl", "trt", "ctrl", "trt", "ctrl"],
    "X": [1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0],
})
```

For this dataset, compute the OLS estimates manually:
- Design matrix: [1, X, D.trt, DX.trt] (10 x 4)
- beta_hat = (X'X)^{-1} X'Y
- Then compute TE(x) = beta_D + beta_DX * x
- Verify Python output matches these manual OLS estimates within 1e-10.

This test validates that the model fitting and TE computation pipeline is correct end-to-end.

#### Test 2.7.2: Manual OLS Verification

For any fixture, independently compute beta_hat = (X'WX)^{-1} X'WY using numpy and compare against `result["model"].params`.

Tolerance: 1e-8.

#### Test 2.7.3: Manual HC1 Verification

For the small cross-reference dataset, compute HC1 variance manually:
```
e = Y - X @ beta_hat
HC1 = n/(n-K) * (X'X)^{-1} @ X' @ diag(e^2) @ X @ (X'X)^{-1}
```
Compare against the vcov used by the package.

Tolerance: 1e-8.

#### Test 2.7.4: Manual Cluster Vcov Verification

For Fixture D (clustered data), compute cluster-robust vcov manually:
```
dfc = (M/(M-1)) * ((N-1)/(N-K))
For each cluster j: u_j = sum_{i in j} X_i * e_i
meat = (1/N) * sum_j u_j u_j'
vcov = dfc * bread @ meat @ bread
```
Compare against the package's cluster vcov output.

Tolerance: 1e-6.

---

## 3. Validation Commands

```bash
# Install the package in development mode
pip install -e .

# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ -v --cov=interflex --cov-report=term-missing

# Run specific test files
pytest tests/test_basic.py -v
pytest tests/test_variance.py -v
pytest tests/test_effects.py -v
```

---

## 4. Test Organization Summary

| File | Tests | What It Validates |
|------|-------|-------------------|
| `test_basic.py` | 2.2.1 - 2.2.10, 2.4.4 - 2.4.5, 2.5.1 - 2.5.10 | Entry point, input validation, treatment detection, basic output structure |
| `test_variance.py` | 2.3.1 - 2.3.10, 2.7.3 - 2.7.4 | Delta/simu/bootstrap SE correctness, vcov types, CI coverage |
| `test_effects.py` | 2.4.1 - 2.4.12, 2.6.1 - 2.6.6, 2.7.1 - 2.7.2 | TE/ME computation, ATE/AME, diff estimates, invariant properties, cross-references |

---

## 5. Tolerance Guidelines

| Category | Tolerance |
|----------|-----------|
| Point estimates vs true DGP values (n=200) | +/- 1.5 (sampling noise) |
| Point estimates vs true DGP values (n=5000) | +/- 0.3 |
| SE comparison between methods (same data) | Within factor of 2 |
| Manual OLS vs package OLS | 1e-8 |
| Manual vcov vs package vcov | 1e-6 (cluster), 1e-8 (HC1, homo) |
| Property invariants (linearity, additivity) | 1e-10 |
| CI coverage across replications | [0.85, 1.00] for n=200 |
| Delta vs simu SE agreement (large n) | Within 15% relative |
