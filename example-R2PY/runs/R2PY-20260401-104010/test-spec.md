# Test Specification: interflex Python Package

## 1. Behavioral Contract

The `interflex` Python package must:

1. Accept a pandas DataFrame plus string column names for Y, D, X, Z, FE, IV, cl, time, weights
2. Validate all inputs and raise `ValueError` with descriptive messages for invalid inputs
3. Fit GLM models (linear, logit, probit, poisson, nbinom) with optional fixed effects and instrumental variables
4. Compute treatment effects (TE) or marginal effects (ME) at a grid of moderator evaluation points
5. Compute predicted values and link values at the same evaluation points
6. Compute differences of TE/ME across specified moderator values
7. Compute average treatment effects (ATE) or average marginal effects (AME)
8. Compute standard errors via three methods: delta method, simulation, bootstrap
9. Compute pointwise and uniform confidence intervals
10. Return all results in an `InterflexResult` dataclass
11. Produce matplotlib plots when `figure=True`
12. Provide a `predict()` method on the result

## 2. Test Scenarios

### 2.1 Synthetic Data Generation

All test data is generated with fixed seeds for reproducibility.

#### Recipe A: Simple Discrete Treatment (Binary)

```python
def make_discrete_binary(n=500, seed=42):
    rng = np.random.default_rng(seed)
    X = rng.uniform(-2, 2, n)
    D = rng.choice([0, 1], n, p=[0.5, 0.5]).astype(str)
    Z1 = rng.normal(0, 1, n)
    # True DGP: Y = 1 + 0.5*X + 2*(D=="1") + 1.5*(D=="1")*X + 0.3*Z1 + eps
    D_num = (D == "1").astype(float)
    eps = rng.normal(0, 0.5, n)
    Y = 1 + 0.5*X + 2*D_num + 1.5*D_num*X + 0.3*Z1 + eps
    return pd.DataFrame({"Y": Y, "D": D, "X": X, "Z1": Z1})
```

True parameters: intercept=1, beta_X=0.5, gamma_1=2, delta_1=1.5, theta_Z1=0.3.
True TE(x) = 2 + 1.5*x.

#### Recipe B: Simple Continuous Treatment

```python
def make_continuous(n=500, seed=42):
    rng = np.random.default_rng(seed)
    X = rng.uniform(-2, 2, n)
    D = rng.normal(0, 1, n)
    eps = rng.normal(0, 0.5, n)
    # True DGP: Y = 1 + 0.5*X + 0.8*D + 0.6*D*X + eps
    Y = 1 + 0.5*X + 0.8*D + 0.6*D*X + eps
    return pd.DataFrame({"Y": Y, "D": D, "X": X})
```

True ME(x) = 0.8 + 0.6*x.

#### Recipe C: Panel Data with Fixed Effects

```python
def make_panel_fe(n_units=50, n_periods=10, seed=42):
    rng = np.random.default_rng(seed)
    n = n_units * n_periods
    unit = np.repeat(np.arange(n_units), n_periods)
    period = np.tile(np.arange(n_periods), n_units)
    unit_fe = rng.normal(0, 2, n_units)
    period_fe = rng.normal(0, 1, n_periods)
    X = rng.uniform(-2, 2, n)
    D = rng.choice(["0", "1"], n, p=[0.5, 0.5])
    D_num = (D == "1").astype(float)
    eps = rng.normal(0, 0.5, n)
    Y = unit_fe[unit] + period_fe[period] + 0.5*X + 2*D_num + 1.5*D_num*X + eps
    return pd.DataFrame({"Y": Y, "D": D, "X": X, "unit": unit, "period": period})
```

#### Recipe D: Multi-arm Discrete Treatment (3 arms)

```python
def make_discrete_multi(n=600, seed=42):
    rng = np.random.default_rng(seed)
    X = rng.uniform(-2, 2, n)
    D = rng.choice(["A", "B", "C"], n, p=[1/3, 1/3, 1/3])
    eps = rng.normal(0, 0.5, n)
    # Base = "A". TE_B(x) = 1 + 0.5*x, TE_C(x) = 3 + 2*x
    Y = 1 + 0.5*X + eps
    Y[D == "B"] += 1 + 0.5*X[D == "B"]
    Y[D == "C"] += 3 + 2*X[D == "C"]
    return pd.DataFrame({"Y": Y, "D": D, "X": X})
```

#### Recipe E: Binary Outcome (Logit/Probit)

```python
def make_binary_outcome(n=1000, seed=42):
    rng = np.random.default_rng(seed)
    X = rng.uniform(-1, 1, n)
    D = rng.choice(["0", "1"], n)
    D_num = (D == "1").astype(float)
    eta = -0.5 + 0.3*X + 1.0*D_num + 0.5*D_num*X
    p = 1 / (1 + np.exp(-eta))
    Y = rng.binomial(1, p, n)
    return pd.DataFrame({"Y": Y, "D": D, "X": X})
```

#### Recipe F: Count Outcome (Poisson)

```python
def make_count_outcome(n=1000, seed=42):
    rng = np.random.default_rng(seed)
    X = rng.uniform(0, 2, n)
    D = rng.choice(["0", "1"], n)
    D_num = (D == "1").astype(float)
    eta = 0.5 + 0.3*X + 0.5*D_num + 0.2*D_num*X
    mu = np.exp(eta)
    Y = rng.poisson(mu, n)
    return pd.DataFrame({"Y": Y, "D": D, "X": X})
```

#### Recipe G: Clustered Data

```python
def make_clustered(n_clusters=30, cluster_size=20, seed=42):
    rng = np.random.default_rng(seed)
    n = n_clusters * cluster_size
    cl = np.repeat(np.arange(n_clusters), cluster_size)
    cluster_effect = rng.normal(0, 1, n_clusters)
    X = rng.uniform(-2, 2, n)
    D = rng.choice(["0", "1"], n)
    D_num = (D == "1").astype(float)
    eps = rng.normal(0, 0.3, n) + cluster_effect[cl]
    Y = 1 + 0.5*X + 2*D_num + 1.5*D_num*X + eps
    return pd.DataFrame({"Y": Y, "D": D, "X": X, "cl": cl})
```

#### Recipe H: Panel Data for PCSE

```python
def make_panel_pcse(n_units=30, n_periods=20, seed=42):
    rng = np.random.default_rng(seed)
    n = n_units * n_periods
    unit = np.repeat(np.arange(n_units), n_periods)
    period = np.tile(np.arange(n_periods), n_units)
    X = rng.uniform(-2, 2, n)
    D = rng.normal(0, 1, n)  # continuous treatment
    eps = rng.normal(0, 0.5, n)
    Y = 1 + 0.5*X + 0.8*D + 0.6*D*X + eps
    return pd.DataFrame({"Y": Y, "D": D, "X": X, "unit": unit, "period": period})
```

#### Recipe I: IV Data

```python
def make_iv_data(n=500, seed=42):
    rng = np.random.default_rng(seed)
    X = rng.uniform(-2, 2, n)
    W = rng.normal(0, 1, n)  # instrument
    # D is endogenous: correlated with error
    v = rng.normal(0, 1, n)
    D_num = 0.5*W + 0.3*W*X + v
    D = (D_num > 0).astype(str)  # make discrete for simplicity
    eps = 0.5*v + rng.normal(0, 0.3, n)  # correlated with v
    D_dum = (D == "True").astype(float)
    Y = 1 + 0.5*X + 2*D_dum + 1.5*D_dum*X + eps
    return pd.DataFrame({"Y": Y, "D": D, "X": X, "W": W})
```

### 2.2 Core Numerical Tests

#### Test N1: Linear Discrete — Coefficient Recovery

**Data**: Recipe A (n=500, seed=42)
**Call**: `interflex("linear", data, "Y", "D", "X", Z=["Z1"], method="linear", vartype="delta", vcov_type="robust")`
**Verify**:
- Estimated TE at x=0 is within 0.3 of true value 2.0
- Estimated TE at x=1 is within 0.3 of true value 3.5
- TE is monotonically increasing with x (since delta_1=1.5 > 0)
- All SEs are positive and finite
- 95% CI at x=0 contains true TE=2.0

**Tolerance**: coefficient estimates within 0.3 of true values (finite sample, n=500)

#### Test N2: Linear Continuous — ME Recovery

**Data**: Recipe B (n=500, seed=42)
**Call**: `interflex("linear", data, "Y", "D", "X", treat_type="continuous", method="linear", vartype="delta")`
**Verify**:
- ME at x=0 approximately 0.8 (within 0.2)
- ME at x=1 approximately 1.4 (within 0.2)
- ME is linear in x
- SEs are positive

**Tolerance**: 0.2 for point estimates

#### Test N3: Fixed Effects — TE Recovery

**Data**: Recipe C (n=500, seed=42)
**Call**: `interflex("linear", data, "Y", "D", "X", FE=["unit", "period"], method="linear", vartype="delta")`
**Verify**:
- TE at x=0 approximately 2.0 (within 0.3)
- TE at x=1 approximately 3.5 (within 0.3)
- `result.use_fe == True`
- Predicted values are all 0 (FE case)

#### Test N4: Multi-arm Discrete — Per-arm TE

**Data**: Recipe D (n=600, seed=42)
**Call**: `interflex("linear", data, "Y", "D", "X", base="A", method="linear", vartype="delta")`
**Verify**:
- `est_lin` has keys for treatment arms "B" and "C" (excluding base "A")
- TE_B at x=0 approximately 1.0 (within 0.3)
- TE_C at x=0 approximately 3.0 (within 0.5)
- Each arm has its own TE, pred, link, diff, ATE entries

#### Test N5: Logit Method — TE Bounds

**Data**: Recipe E (n=1000, seed=42)
**Call**: `interflex("linear", data, "Y", "D", "X", method="logit", vartype="delta")`
**Verify**:
- All TE values are in (-1, 1) (bounded by probability difference)
- All predicted values are in (0, 1)
- SEs are positive

#### Test N6: Poisson Method — Non-negative Predictions

**Data**: Recipe F (n=1000, seed=42)
**Call**: `interflex("linear", data, "Y", "D", "X", method="poisson", vartype="delta")`
**Verify**:
- All predicted values are > 0 (exp link ensures positivity)
- SEs are positive

### 2.3 Variance Type Tests

#### Test V1: Simulation Variance — SD Consistency

**Data**: Recipe A (n=500, seed=42)
**Call with vartype="simu"** and **vartype="delta"**
**Verify**:
- Simulation SDs and delta SDs agree within relative tolerance of 0.3 (they should be similar for well-behaved linear models)
- Both produce valid CIs (lower < estimate < upper)

#### Test V2: Bootstrap Variance — SD Consistency

**Data**: Recipe A (n=200, seed=42, small n for speed)
**Call with vartype="bootstrap", nboots=100**
**Verify**:
- Bootstrap SDs are positive and finite
- Bootstrap CIs contain the point estimate
- Bootstrap SDs are in the same order of magnitude as delta SDs

#### Test V3: Delta Method — Analytical Properties

**Data**: Recipe B continuous (n=500)
**Call with vartype="delta", method="linear"**
**Verify**:
- For linear continuous: SE of ME depends only on (beta_D, beta_DX) part of vcov
- SE at x=0 equals sqrt(V[D,D]) (since ME(0) = beta_D, gradient = (0,0,1,0,...))
- SE increases as |x| increases (due to DX term contribution)

#### Test V4: Simulation Variance — Reproducibility

**Call same setup twice with same seed via `np.random.default_rng(seed)`**
**Verify**: Results are identical

### 2.4 Vcov Type Tests

#### Test C1: Homoscedastic — Smaller than Robust

**Data**: Recipe A with heteroscedastic error: `eps = rng.normal(0, 0.1 + abs(X), n)`
**Call with vcov_type="homoscedastic" and vcov_type="robust"**
**Verify**:
- Both produce valid vcov matrices (symmetric, positive semi-definite)
- Robust SEs are generally larger than homoscedastic for this DGP

#### Test C2: Clustered SE — Larger than Robust

**Data**: Recipe G (clustered data)
**Call with vcov_type="cluster", cl="cl"** and **vcov_type="robust"**
**Verify**:
- Clustered SEs are generally larger than robust SEs (due to within-cluster correlation)
- Cluster vcov is symmetric and PSD
- DFC factor is applied: (M/(M-1)) * ((N-1)/(N-K))

#### Test C3: PCSE — Panel Corrected SE

**Data**: Recipe H (panel data)
**Call with vcov_type="pcse", cl="unit", time="period"**
**Verify**:
- PCSE vcov is symmetric and PSD
- PCSE is only available for method="linear"
- PCSE produces valid standard errors (positive, finite)

#### Test C4: Vcov Symmetry Fallback

**Verify**: If vcov is not symmetric within tolerance 1e-6, the code falls back to homoscedastic vcov and issues a warning.

### 2.5 FWL Demeaning Tests

#### Test F1: Single FE — Correct Demeaning

**Data**: n=100, 10 groups, known group means
**Verify**:
- After demeaning, within-group means of Y and each X column are approximately 0 (tolerance 1e-4)
- Number of iterations < 50

#### Test F2: Two FEs — Convergence

**Data**: Recipe C (two-way FE)
**Verify**:
- Convergence within 50 iterations
- Coefficients match OLS on manually demeaned data (using iterative projection)
- Residuals satisfy sum(residuals * weight) approx 0

#### Test F3: FWL OLS Equivalence

**Data**: Single FE, small dataset where we can compute the answer manually
**Verify**:
- FWL + OLS gives same coefficients as LSDV (least squares dummy variable regression) within tolerance 1e-6

#### Test F4: IV FWL — 2SLS Correctness

**Data**: Recipe I with FE
**Verify**:
- `iv_fwl()` produces coefficients
- Residuals have correct dimension
- Grand mean is finite

### 2.6 ATE/AME Tests

#### Test A1: ATE Discrete Linear

**Data**: Recipe A
**Verify**:
- ATE is approximately `mean(TE(X_i))` for treated observations = `mean(2 + 1.5*X_i)` for D=="1" group
- ATE is finite, has positive SE
- ATE z-value = ATE/SE, p-value = 2*Phi(-|z|)

#### Test A2: AME Continuous Linear

**Data**: Recipe B
**Verify**:
- AME = mean(0.8 + 0.6*X_i) across all observations
- AME approximately 0.8 (since E[X]=0 for uniform[-2,2])
- AME SE is positive

#### Test A3: ATE with Delta SE

**Data**: Recipe A
**Call with delta=True**
**Verify**:
- ATE and ATE SE are both returned
- CI is (ATE - crit*SE, ATE + crit*SE) for delta
- CI is finite and valid (lower < upper)

### 2.7 Uniform CI Tests

#### Test U1: Bootstrap Uniform CI — Coverage Property

**Data**: Recipe A, vartype="bootstrap", nboots=200
**Verify**:
- Uniform CI is wider than (or equal to) pointwise CI at every evaluation point
- Uniform CI result has 7 columns: X, TE, sd, lower, upper, uniform_lower, uniform_upper
- uniform_lower <= lower and uniform_upper >= upper at every point

#### Test U2: Delta Uniform CI — Width Property

**Data**: Recipe A, vartype="delta"
**Verify**:
- Uniform CI critical value q > normal critical value (approx 1.96)
- Uniform CI is wider than pointwise CI at every evaluation point

#### Test U3: Uniform Quantiles — Bonferroni Bound

**Verify**: For very few bootstrap samples (e.g., N=10), the function should warn and fall back to Bonferroni: zeta = alpha/(2k).

### 2.8 Difference Tests

#### Test D1: Two-point Difference

**Data**: Recipe A
**Call with diff_values=[Q25_X, Q75_X]**
**Verify**:
- diff_estimate = TE(Q75) - TE(Q25)
- For linear TE(x) = 2 + 1.5x: diff = 1.5*(Q75 - Q25)
- diff has SD, z-value, p-value, CI
- z-value = diff/SD

#### Test D2: Three-point Difference

**Data**: Recipe A
**Call with diff_values=[Q25, Q50, Q75]**
**Verify**:
- Three differences returned: (Q50-Q25, Q75-Q50, Q75-Q25)
- Each has SD, z-value, p-value, CI
- Third difference equals sum of first two (numerically)

### 2.9 Input Validation Tests

#### Test I1: Missing Column Raises ValueError

```python
with pytest.raises(ValueError):
    interflex("linear", data, "NONEXISTENT", "D", "X")
```

#### Test I2: Invalid Method Raises ValueError

```python
with pytest.raises(ValueError):
    interflex("linear", data, "Y", "D", "X", method="invalid")
```

#### Test I3: FE with Non-linear Method Raises ValueError

```python
with pytest.raises(ValueError):
    interflex("linear", data, "Y", "D", "X", FE=["unit"], method="logit")
```

#### Test I4: Cluster Without cl Raises ValueError

```python
with pytest.raises(ValueError):
    interflex("linear", data, "Y", "D", "X", vcov_type="cluster")
```

#### Test I5: PCSE Requires cl and time

```python
with pytest.raises(ValueError):
    interflex("linear", data, "Y", "D", "X", vcov_type="pcse")
```

#### Test I6: Binary Outcome for Logit

```python
# Y with values not in {0,1}
data_bad = pd.DataFrame({"Y": [0, 1, 2], "D": ["a","a","b"], "X": [1,2,3]})
with pytest.raises(ValueError):
    interflex("linear", data_bad, "Y", "D", "X", method="logit")
```

#### Test I7: NA Handling

```python
# Data with NAs, na_rm=False should raise
data_na = make_discrete_binary()
data_na.loc[0, "Y"] = np.nan
with pytest.raises(ValueError):
    interflex("linear", data_na, "Y", "D", "X", na_rm=False)
# na_rm=True should succeed
result = interflex("linear", data_na, "Y", "D", "X", na_rm=True)
assert result is not None
```

#### Test I8: diff_values Length

```python
with pytest.raises(ValueError):
    interflex("linear", data, "Y", "D", "X", diff_values=np.array([1.0]))  # length 1
```

### 2.10 InterflexResult Structure Tests

#### Test R1: Result Has All Expected Fields

```python
result = interflex("linear", data, "Y", "D", "X", ...)
assert hasattr(result, "est_lin")
assert hasattr(result, "pred_lin")
assert hasattr(result, "link_lin")
assert hasattr(result, "diff_estimate")
assert hasattr(result, "vcov_matrix")
assert hasattr(result, "avg_estimate")
assert hasattr(result, "treat_info")
assert hasattr(result, "estimator")
assert result.estimator == "linear"
```

#### Test R2: est_lin Structure

```python
# Discrete binary: one treatment arm
result = interflex("linear", data, "Y", "D", "X", base="0")
assert len(result.est_lin) == 1  # one non-base arm
key = list(result.est_lin.keys())[0]
te_table = result.est_lin[key]
assert te_table.shape[1] >= 5  # X, TE, sd, lower, upper (+ optional uniform)
assert te_table.shape[0] > 0  # at least one evaluation point
```

#### Test R3: Predict Method Works

```python
result = interflex("linear", data, "Y", "D", "X", figure=False)
fig = result.predict(type="response")
# For non-FE: returns a Figure
# For FE: returns None
```

### 2.11 Plotting Tests

#### Test P1: Figure Is Created

```python
result = interflex("linear", data, "Y", "D", "X", figure=True)
assert result.figure is not None
assert isinstance(result.figure, matplotlib.figure.Figure)
```

#### Test P2: Pool Plot

```python
result = interflex("linear", data_multi, "Y", "D", "X", base="A", figure=True, pool=True)
assert result.figure is not None
```

#### Test P3: Save to File

```python
import tempfile, os
with tempfile.NamedTemporaryFile(suffix=".png", delete=False) as f:
    fname = f.name
result = interflex("linear", data, "Y", "D", "X", figure=True, file=fname)
assert os.path.exists(fname)
os.unlink(fname)
```

### 2.12 Edge Case Tests

#### Test E1: Single Treatment Arm (No Other Treat)

**Data**: All D == "A" (only one group)
**Expected**: ValueError — need at least two treatment arms for discrete

#### Test E2: Very Small Sample

**Data**: n=20, discrete binary
**Verify**: Model fits without error, but SEs may be large

#### Test E3: Perfect Collinearity

**Data**: Z1 = 2*Z2 (perfectly collinear covariates)
**Verify**: Model handles gracefully (drops one, sets coef to NaN/0)

#### Test E4: Zero-Variance Moderator After FE Demeaning

**Data**: X is constant within each FE group
**Verify**: After FWL, X has no variance. Coefficient set to NaN, code handles gracefully.

#### Test E5: All Weights Equal to 1

**Data**: Recipe A with weights=None
**Verify**: Same results as explicitly setting all weights to 1

#### Test E6: Continuous Treatment Auto-Detection

**Data**: D has 10 unique numeric values
**Verify**: `treat_type` auto-detected as "continuous"

#### Test E7: Discrete Treatment Auto-Detection

**Data**: D has 3 unique values
**Verify**: `treat_type` auto-detected as "discrete"

## 3. Test Matrix

### Method x VarType x VcovType Combinations

| method | vartype | vcov_type | treat_type | Data Recipe | Priority |
|--------|---------|-----------|------------|-------------|----------|
| linear | delta | robust | discrete | A | HIGH |
| linear | delta | homoscedastic | discrete | A | HIGH |
| linear | delta | cluster | discrete | G | HIGH |
| linear | delta | pcse | continuous | H | MEDIUM |
| linear | simu | robust | discrete | A | HIGH |
| linear | simu | robust | continuous | B | HIGH |
| linear | bootstrap | robust | discrete | A | HIGH |
| linear | bootstrap | cluster | discrete | G | MEDIUM |
| logit | delta | robust | discrete | E | HIGH |
| logit | simu | robust | discrete | E | MEDIUM |
| probit | delta | robust | discrete | E | MEDIUM |
| poisson | delta | robust | discrete | F | MEDIUM |
| nbinom | delta | robust | discrete | F | LOW |
| linear | delta | robust | discrete+FE | C | HIGH |
| linear | delta | cluster | discrete+FE | C | MEDIUM |
| linear | delta | robust | discrete+IV | I | MEDIUM |

### Feature Test Matrix

| Feature | Test IDs | Priority |
|---------|----------|----------|
| Input validation | I1-I8 | HIGH |
| Linear discrete TE | N1, N4 | HIGH |
| Linear continuous ME | N2 | HIGH |
| Fixed effects | N3, F1-F4 | HIGH |
| Logit/probit TE bounds | N5 | HIGH |
| Poisson predictions | N6 | MEDIUM |
| Delta method SE | V3, V4 | HIGH |
| Simulation variance | V1 | HIGH |
| Bootstrap variance | V2 | MEDIUM |
| Clustered SE | C2 | HIGH |
| Robust SE | C1 | HIGH |
| PCSE | C3 | MEDIUM |
| ATE/AME | A1-A3 | HIGH |
| Uniform CI | U1-U3 | MEDIUM |
| Differences | D1-D2 | MEDIUM |
| Result structure | R1-R3 | HIGH |
| Plotting | P1-P3 | LOW |
| Edge cases | E1-E7 | MEDIUM |

## 4. Tolerance Levels

| Quantity | Tolerance | Rationale |
|----------|-----------|-----------|
| Coefficient recovery (n=500) | abs < 0.3 | Finite sample with noise; 3-sigma range |
| TE/ME point estimates | abs < 0.3 | Same as above |
| Delta vs simulation SD ratio | within factor of 2 | Simulation variance converges slowly |
| Bootstrap vs delta SD ratio | within factor of 3 | Bootstrap with nboots=100 is noisy |
| FWL demeaning residual group means | abs < 1e-4 | Convergence tolerance is 1e-5 on total |
| FWL vs LSDV coefficient match | abs < 1e-6 | Should be numerically identical |
| Vcov symmetry | max asymmetry < 1e-6 | Matches R check |
| CI validity | lower < estimate < upper | Structural property |
| Uniform CI vs pointwise CI | uniform width >= pointwise width | By construction |
| TE bounds (logit) | -1 < TE < 1 | Probability differences |
| Predictions (poisson) | > 0 | Exp link positivity |

## 5. Validation Commands

```bash
# Run all tests
pytest tests/ -v --tb=short

# Run with coverage
pytest tests/ -v --cov=interflex --cov-report=term-missing

# Run specific test file
pytest tests/test_linear.py -v

# Run only HIGH priority tests (via markers)
pytest tests/ -v -m "high_priority"

# Quick smoke test
pytest tests/test_core.py tests/test_linear.py -v --tb=short -x
```

## 6. Property-Based Invariants

1. **TE linearity (linear method, discrete)**: TE(x) = gamma + delta*x must be exactly linear in x. Check: fit a line to (X_eval, TE) — R-squared must be > 0.9999.

2. **ME linearity (linear method, continuous)**: ME(x) = beta_D + beta_DX*x must be exactly linear. Same check.

3. **Prediction consistency**: For linear method without FE: E_pred(x) = link(x). For logit: E_pred = expit(link).

4. **Difference additivity**: For 3-point differences: diff(x3,x1) = diff(x2,x1) + diff(x3,x2). Tolerance: 1e-10.

5. **ATE/AME weighted mean property**: ATE should approximately equal the weighted average of TE(X_i) over the treatment group. Tolerance: 1e-10 (numerical).

6. **Bootstrap CI percentile property**: lower = 2.5th percentile, upper = 97.5th percentile of bootstrap distribution. Verify by checking that approximately 95% of bootstrap replicates fall within (lower, upper).

7. **Delta CI symmetry (linear)**: For linear method, CI is symmetric around the point estimate (since SE formula is symmetric). Check: `abs((upper - estimate) - (estimate - lower)) < 1e-10`.

8. **Vcov positive semi-definiteness**: All eigenvalues of model_vcov >= -1e-10.

9. **FE prediction is zero**: When `use_fe=True`, all predicted values and link SDs must be exactly 0.

10. **Weight invariance**: Multiplying all weights by a constant c > 0 should not change coefficients or TE/ME estimates (only variance estimates scale).
