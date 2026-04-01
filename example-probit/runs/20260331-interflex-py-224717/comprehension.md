# Comprehension Verification

## Date
2026-03-31

## HOLD Rounds Used
0

## Input Materials Read

1. **R/linear.R** (2249 lines) — Core linear estimator `interflex.linear()` containing: formula construction, model fitting, treatment effect computation (`gen.general.TE`), delta method variance (`gen.delta.TE`), ATE computation (`gen.ATE`), simulation variance (Part 1), delta variance (Part 2), bootstrap variance (Part 3), and return structure assembly.
2. **R/interflex.R** (lines 1-950) — Entry point `interflex()` with: input validation for all parameters, missing value handling, covariate preprocessing (factor to dummy conversion via `contr.sum`), treatment type detection (discrete if <= 5 unique values or non-numeric, else continuous), treatment group renaming to `Group.1`, `Group.2`, etc., moderator quantile difference setup, `Z.X` interaction term creation for fully moderated models, `Z.ref` defaulting to covariate means, and routing to estimator-specific functions.
3. **R/vcluster.R** (39 lines) — Cluster-robust variance-covariance estimator implementing the sandwich formula with the Cameron-Gelbach-Miller (CGM) small-sample degrees-of-freedom correction.
4. **request.md** — Scope definition for the Python translation.
5. **impact.md** — Affected surfaces and risk areas.

## Restated Core Requirement

Translate the linear estimator path from R's interflex package into a standalone Python package. The package estimates heterogeneous treatment/marginal effects as a function of a moderator variable X, supporting both discrete treatments (binary or multi-valued categorical D) and continuous treatments. The model is Y = intercept + beta_X * X + sum_j(beta_Dj * D_j + beta_DXj * D_j * X) + Z'gamma [+ Z'delta * X if fully moderated], fitted by OLS (or WLS if weights provided). Treatment effects TE(x) or marginal effects ME(x) are computed at an evaluation grid of X values, with standard errors from one of three methods: delta method, simulation draws from N(beta_hat, Sigma_hat), or nonparametric bootstrap. Cluster-robust, HC1, and homoscedastic variance-covariance options are available. Average treatment effects (ATE/AME) are also computed. Difference estimates at moderator quantiles compare effects at different X values. The Python package must NOT include fixed effects, instrumental variables, PCSE, non-linear link functions, plotting, or uniform CIs.

## Formulas Restated and Verified

### Model Specification

**Discrete treatment** (D is categorical with J+1 groups, group `base` is reference):

For each non-base group j, create indicators:
- `D.j = 1{D == j}`
- `DX.j = D.j * X`

Full model (without full.moderate):
```
Y = beta_0 + beta_X * X + sum_{j != base}(beta_Dj * D.j + beta_DXj * DX.j) + sum_k(gamma_k * Z_k) + epsilon
```

With full.moderate (Z x X interactions):
```
Y = beta_0 + beta_X * X + sum_{j != base}(beta_Dj * D.j + beta_DXj * DX.j) + sum_k(gamma_k * Z_k + delta_k * Z_k * X) + epsilon
```

**Continuous treatment**:
```
Y = beta_0 + beta_X * X + beta_D * D + beta_DX * D * X + sum_k(gamma_k * Z_k) [+ sum_k(delta_k * Z_k * X)] + epsilon
```

### Treatment Effect / Marginal Effect (gen.general.TE)

**Discrete, linear method** at evaluation point x:
```
link_1(x) = beta_0 + beta_X * x + beta_Dj + beta_DXj * x + sum_k(gamma_k * Z_ref_k) [+ sum_k(delta_k * Z_ref_k * x)]
link_0(x) = beta_0 + beta_X * x + sum_k(gamma_k * Z_ref_k) [+ sum_k(delta_k * Z_ref_k * x)]
TE_j(x) = link_1(x) - link_0(x) = beta_Dj + beta_DXj * x
```

Note: In the linear case, covariates Z cancel out in the TE, so TE_j(x) = beta_Dj + beta_DXj * x regardless of Z or full.moderate. But the predicted values E.pred and E.base DO depend on Z.ref.

**Continuous, linear method** at evaluation point x, reference D value D_ref:
```
link(x) = beta_0 + beta_X * x + beta_D * D_ref + beta_DX * x * D_ref + sum_k(gamma_k * Z_ref_k) [+ sum_k(delta_k * Z_ref_k * x)]
ME(x) = dY/dD = beta_D + beta_DX * x
E.pred(x) = link(x)
```

### Difference Estimates

Given diff.values = [x1, x2] or [x1, x2, x3]:
- If 2 values: diff = TE(x2) - TE(x1)
- If 3 values: diffs = [TE(x2)-TE(x1), TE(x3)-TE(x2), TE(x3)-TE(x1)]

Default diff.values = quantile(X, [0.25, 0.5, 0.75]) with names "50% vs 25%", "75% vs 50%", "75% vs 25%".

### Delta Method Variance (gen.delta.TE)

**Discrete, linear, no FE, no Z**:
```
vec_1 = [1, x, 1, x]  (gradient of link_1 w.r.t. [intercept, beta_X, beta_Dj, beta_DXj])
vec_0 = [1, x, 0, 0]
vec = vec_1 - vec_0 = [0, 0, 1, x]
target_slice = ["(Intercept)", X, "D.j", "DX.j"]
Sigma = model.vcov[target_slice, target_slice]
sd_TE(x) = sqrt(vec' * Sigma * vec)
```

**Discrete, linear, no FE, with Z (no full.moderate)**:
```
vec_1 = [1, x, 1, x, Z_ref]
vec_0 = [1, x, 0, 0, Z_ref]
vec = vec_1 - vec_0 = [0, 0, 1, x, 0, ..., 0]
target_slice = ["(Intercept)", X, "D.j", "DX.j", Z...]
```

**Discrete, linear, no FE, with Z and full.moderate**:
```
vec_1 = [1, x, 1, x, Z_ref, x*Z_ref]
vec_0 = [1, x, 0, 0, Z_ref, x*Z_ref]
vec = vec_1 - vec_0 = [0, 0, 1, x, 0, ..., 0, 0, ..., 0]
target_slice = ["(Intercept)", X, "D.j", "DX.j", Z..., Z.X...]
```

**Continuous, linear**:
```
vec = [0, 0, 1, x]  (gradient of ME w.r.t. [intercept, X, D, DX])
target_slice = ["(Intercept)", X, D, "DX"]
sd_ME(x) = sqrt(vec' * Sigma * vec)
```

With Z: vec = [0, 0, 1, x, 0, ..., 0]

**Difference SE** (delta method):
```
vec_diff = vec(x2) - vec(x1)
sd_diff = sqrt(vec_diff' * Sigma * vec_diff)
```

**CI using delta** (t-distribution):
```
crit = |t_{0.025, df}|
CI = [estimate - crit * sd, estimate + crit * sd]
```

### Predicted Value SE (delta method)

**Discrete, non-base, linear**:
```
vec = [1, x, 1, x]  (optionally + Z_ref, Z_ref*x)
sd_predict = sqrt(vec' * Sigma * vec)
```

**Discrete, base group**:
```
vec = [1, x]  (optionally + Z_ref, Z_ref*x)
```

### Simulation Variance (vartype="simu")

1. Draw M = nsimu coefficient vectors from N(beta_hat, Sigma_hat) using `rmvnorm`
2. For each draw, compute TE(x) / ME(x) at all evaluation points
3. sd = sample SD across draws
4. CI = [2.5th percentile, 97.5th percentile] across draws
5. vcov = sample covariance matrix across draws

For ATE (discrete): also compute ATE for each draw and take SD/quantiles.
For AME (continuous): also compute AME for each draw and take SD/quantiles.

### Bootstrap Variance (vartype="bootstrap")

1. For b = 1, ..., nboots:
   a. If no clustering: sample n observations with replacement
   b. If clustering: sample clusters with replacement (block bootstrap), take all obs in sampled clusters
   c. Refit model on bootstrap sample
   d. Compute TE/ME, predictions, differences, ATE/AME using bootstrap coefficients
2. sd = sample SD across bootstrap replicates
3. CI = [2.5th percentile, 97.5th percentile]
4. vcov = sample covariance across replicates

### ATE/AME (gen.ATE)

**Discrete, linear**:
```
For each observation i in treatment group j:
  TE_i = beta_Dj + X_i * beta_DXj  (simplified for linear; full formula uses link differences)
ATE_j = weighted.mean(TE_i, weights_i)
```

**Continuous, linear**:
```
For each observation i:
  ME_i = beta_D + beta_DX * X_i
AME = weighted.mean(ME_i, weights_i)
```

**ATE/AME SE (delta method)**:
```
For each observation i, compute gradient vector vec_i
vec_mean = weighted.mean(vec_i across observations)
sd = sqrt(vec_mean' * Sigma * vec_mean)
```

### Cluster-Robust Variance (vcluster.R)

```
M = number of unique clusters
N = number of observations
K = model rank (number of estimated coefficients)
dfc = (M/(M-1)) * ((N-1)/(N-K))   # CGM small-sample correction

u_j = sum of estimating equations within cluster j  (tapply sum by cluster)
V_cluster = dfc * bread(model) * (sum_j u_j u_j') / N * bread(model)
         = dfc * sandwich(model, bread=bread, meat=crossprod(uj)/N)
```

### Evaluation Grid

```
X.eval = linspace(min(X), max(X), neval)   # neval default = 50
X.eval = sort(union(X.eval, X.eval0))      # merge with user-specified points
```

### Treatment Type Detection (interflex.R)

```
if treat.type is NULL:
  if D is numeric AND has > 5 unique values: treat.type = "continuous"
  else: treat.type = "discrete"
```

### Treatment Renaming (interflex.R)

Discrete treatments are renamed to Group.1, Group.2, ... (sorted). The mapping is preserved via `all.treat.origin` (original -> renamed) and reversed for output. The `base` group is the first sorted value by default.

### Covariate Preprocessing (interflex.R)

- Factor covariates: converted to sum-coded dummy variables via `contr.sum`
- Z.ref defaults: mean of numeric covariates, 0 for dummy columns from factors
- Z.X interaction terms: `Z_k * X` for each covariate when `full.moderate=TRUE`

### Output Structure

For discrete: `est.lin` is a dict of {treatment_label: matrix(X, TE, sd, lower_CI, upper_CI)}
For continuous: `est.lin` is a dict of {D_ref_label: matrix(X, ME, sd, lower_CI, upper_CI)}

Both include: `pred.lin`, `diff.estimate`, `vcov.matrix`, `Avg.estimate`

## Undefined Symbols or Ambiguities

None. All symbols, methods, and steps are fully defined in the R source code.

## Judgment Calls Not Supported by Source

1. **Factor covariate handling**: The R code uses `contr.sum` for factor-to-dummy conversion. In Python, we will use a similar sum-coding scheme or simply provide dummy variables. Since the Python API will accept DataFrames, we can detect object/category columns and handle them similarly. Decision: use pandas `get_dummies` with `drop_first=False` and then apply sum coding by setting the last level to -1 for all other dummies. This matches `contr.sum` behavior.

2. **Treatment renaming**: The R code renames treatments to `Group.1`, `Group.2`, etc. to avoid special characters. In Python, we will do the same internal renaming for formula construction but map back to original labels for output.

## Intuition: Why Does This Work?

The interflex linear estimator is fundamentally an interaction model. For discrete treatments, it creates dummy variables for each treatment group and interacts them with the moderator X. The treatment effect at any value of X is then a linear function of X (TE(x) = alpha + beta*x), allowing the effect to vary with X. This is the simplest form of heterogeneous treatment effects.

The three variance estimation methods trade off analytical tractability vs robustness:
- Delta method: fastest, uses analytical gradients, relies on asymptotic normality of beta_hat
- Simulation: draws from the estimated sampling distribution of beta_hat, captures nonlinearity in the TE function of beta (though for linear method, TE is linear in beta, so simulation and delta give similar results)
- Bootstrap: most robust, does not rely on the asymptotic variance estimate, captures finite-sample behavior

## Implicit Assumptions

1. The model is correctly specified (linearity in parameters)
2. OLS assumptions hold (or are relaxed via robust/cluster SEs)
3. The moderator X is a continuous variable (evaluation grid is linspace)
4. Z.ref values are used when computing predicted values and treatment effects that depend on covariates (default: covariate means)
5. For cluster-robust SEs, cluster membership is fixed and clusters are independent
6. Bootstrap validity requires sufficient cluster count for block bootstrap

## Comprehension Verdict

**FULLY UNDERSTOOD** — All formulas verified against source. All algorithms traced step-by-step. No undefined symbols. No ambiguous steps. Ready to produce specs.
