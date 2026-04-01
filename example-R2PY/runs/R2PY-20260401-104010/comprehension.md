# Comprehension Record

## Date
2026-04-01

## HOLD Rounds Used
0

## Input Materials Read

| # | File | Type | Content |
|---|------|------|---------|
| 1 | `R/interflex.R` (1232 lines) | R source | Main entry point: input validation, treat preprocessing, covariate dummy encoding, Z.ref defaults, FE/cl factorization, routing to `interflex.linear()` |
| 2 | `R/linear.R` (2249 lines) | R source | Full linear estimator: formula construction, model fitting (glm/felm/ivreg/glm.nb), vcov computation, **Function A** (gen.general.TE), **Function B** (gen.delta.TE), **Function C** (gen.ATE), variance paths (simu/delta/bootstrap), uniform CI, histogram/density, output assembly |
| 3 | `R/plot.R` (1560 lines) | R source | `plot.interflex()` S3 method: ggplot2 faceted panel plots for TE/ME/predictions with CI ribbons, uniform CI dashed ribbons, histograms/density underneath, pool mode, facet layout |
| 4 | `R/predict.R` (730 lines) | R source | `predict.interflex()` S3 method: extracts predicted values from the fitted object, replots with optional type="link" |
| 5 | `R/vcluster.R` (39 lines) | R source | `vcovCluster()`: clustered sandwich variance using `sandwich::estfun()` and `sandwich::bread()` with DFC = (M/(M-1))((N-1)/(N-K)) |
| 6 | `R/uniform.R` (40 lines) | R source | Two functions: `calculate_uniform_quantiles()` (bootstrap-based bisection for uniform CI) and `calculate_delta_uniformCI()` (MVN simulation for delta-method uniform CI) |
| 7 | `src/fastplm.cpp` (228 lines) | C++ | FWL iterative demeaning for multiple fixed effects with weights; convergence tol=1e-5, max 50 iterations; followed by weighted OLS on demeaned data |
| 8 | `src/iv_fastplm.cpp` (203 lines) | C++ | Same FWL demeaning + 2SLS: after demeaning Y, X(endogenous), Z(exogenous), IV(instruments), computes Pz = Z_full(Z_full'Z_full)^{-1}Z_full', then solves PzX * coef = PzY |
| 9 | `request.md` | Run artifact | Scope definition |
| 10 | `impact.md` | Run artifact | File mapping, package structure, R-to-Python library decisions |

## Restated Core Requirement

Translate the `estimator = "linear"` code path from the R package `interflex` (v1.3.2) into a standalone Python package named `interflex`. The Python package must:

- Accept the same conceptual API: an `interflex()` function taking outcome Y, treatment D, moderator X, covariates Z, fixed effects FE, instruments IV, and many tuning parameters.
- Implement the same model: Y ~ X + D + D*X (+ Z) with optional fixed effects (FWL demeaning) and IV (2SLS after FWL). For discrete treatment, one set of (D_j, DX_j) dummies per non-base treatment arm. For continuous treatment, a single (D, DX) pair.
- Support 5 GLM methods: linear (OLS), logit, probit, poisson, negative binomial.
- Support 4 vcov types: homoscedastic, robust (HC1), cluster, PCSE.
- Support 3 variance inference paths: delta method, simulation (MVN draws from coefficient distribution), nonparametric bootstrap (with block bootstrap for clustered data).
- Compute treatment effects (TE for discrete) or marginal effects (ME for continuous) at evaluation points X.eval, plus predicted values, link values, differences across moderator values, and ATE/AME.
- Compute uniform confidence intervals (both bootstrap-based bisection and delta-method MVN simulation).
- Produce matplotlib/seaborn plots equivalent to the ggplot2 output.
- Provide a predict method.
- Be pip-installable with pyproject.toml.

## Formulas Restated and Verified

### Model Specification

**Discrete treatment** (base group b, treatment arms j != b):

$$Y_i = \alpha + \beta_X X_i + \sum_{j \neq b} (\gamma_j D_{ij} + \delta_j D_{ij} X_i) + Z_i' \theta + \epsilon_i$$

where $D_{ij} = \mathbb{1}(D_i = j)$.

If `full.moderate = TRUE`, add $Z_i \cdot X_i$ interaction terms.

**Continuous treatment**:

$$Y_i = \alpha + \beta_X X_i + \beta_D D_i + \beta_{DX} D_i X_i + Z_i' \theta + \epsilon_i$$

### Treatment Effects at Evaluation Points

**Discrete, linear method**:

$$TE_j(x) = \gamma_j + \delta_j x$$

**Discrete, logit**:

$$TE_j(x) = \frac{\exp(\eta_1(x))}{1+\exp(\eta_1(x))} - \frac{\exp(\eta_0(x))}{1+\exp(\eta_0(x))}$$

where $\eta_1(x) = \alpha + \beta_X x + \gamma_j + \delta_j x + Z_{ref}' \theta$ and $\eta_0(x) = \alpha + \beta_X x + Z_{ref}' \theta$.

**Discrete, probit**:

$$TE_j(x) = \Phi(\eta_1(x)) - \Phi(\eta_0(x))$$

**Discrete, poisson/nbinom**:

$$TE_j(x) = \exp(\eta_1(x)) - \exp(\eta_0(x))$$

**Continuous, linear**:

$$ME(x) = \beta_D + \beta_{DX} x$$

**Continuous, logit**:

$$ME(x) = \frac{\exp(\eta(x))}{(1+\exp(\eta(x)))^2} (\beta_D + \beta_{DX} x)$$

where $\eta(x) = \alpha + \beta_X x + \beta_D D_{ref} + \beta_{DX} x D_{ref} + Z_{ref}'\theta$.

**Continuous, probit**:

$$ME(x) = (\beta_D + \beta_{DX} x) \phi(\eta(x))$$

**Continuous, poisson/nbinom**:

$$ME(x) = \exp(\eta(x)) (\beta_D + \beta_{DX} x)$$

### Fixed Effects (FWL)

For FE models, treatment effects simplify:
- Discrete: $TE_j(x) = \gamma_j + \delta_j x$ (no intercept, no Z, no link transforms needed since FE only works with `method="linear"`)
- Continuous: $ME(x) = \beta_D + \beta_{DX} x$
- Predicted values are 0 (cannot recover level predictions with FE)

### Delta Method SE

For TE/ME SE at evaluation point $x$:

$$SE(x) = \sqrt{g'(x)^T V g'(x)}$$

where $g'(x)$ is the gradient of TE/ME with respect to the coefficient vector $\beta$, and $V$ is the variance-covariance matrix of $\hat\beta$.

**Discrete, linear, no FE**: $g'(x) = (1, x, 1, x, Z_{ref}) - (1, x, 0, 0, Z_{ref}) = (0, 0, 1, x, 0)$ for the relevant coefficient subvector.

**Discrete, linear, FE**: $g'(x) = (1, x)$ on the subvector $(\gamma_j, \delta_j)$.

**Discrete, logit, no FE**: $g'(x) = v_1 \cdot \frac{\exp(\eta_1)}{(1+\exp(\eta_1))^2} - v_0 \cdot \frac{\exp(\eta_0)}{(1+\exp(\eta_0))^2}$

where $v_1 = (1, x, 1, x, Z_{ref})$ and $v_0 = (1, x, 0, 0, Z_{ref})$.

**Continuous, linear, no FE**: $g'(x) = (0, 0, 1, x, 0)$ — only $(\beta_D, \beta_{DX})$ matter.

**Continuous, logit, no FE**: $g'(x) = -(\beta_D + x\beta_{DX})\frac{\exp(\eta)-\exp(-\eta)}{(2+\exp(\eta)+\exp(-\eta))^2} v_1 + \frac{\exp(\eta)}{(1+\exp(\eta))^2} v_0$

where $v_1 = (1, x, D_{ref}, D_{ref} x, Z_{ref})$ and $v_0 = (0, 0, 1, x, 0)$.

### Simulation Variance

1. Draw $M$ coefficient vectors: $\tilde\beta^{(m)} \sim \mathcal{N}(\hat\beta, \hat{V})$
2. For each draw, compute $TE^{(m)}(x)$ or $ME^{(m)}(x)$
3. $SD(x) = \text{sd}(\{TE^{(m)}(x)\}_{m=1}^M)$
4. CI: 2.5th and 97.5th percentiles

### Bootstrap Variance

1. Resample $n$ observations with replacement (block bootstrap if clustered: resample clusters)
2. Refit model on bootstrap sample
3. Compute $TE^{(b)}(x)$ or $ME^{(b)}(x)$
4. $SD(x) = \text{sd}(\{TE^{(b)}(x)\}_{b=1}^B)$
5. CI: 2.5th and 97.5th percentiles

### ATE / AME

**ATE (discrete, no FE)**:

$$ATE_j = \frac{\sum_{i: D_i = j} w_i \cdot TE_j(X_i, Z_i)}{\sum_{i: D_i = j} w_i}$$

where $TE_j(X_i, Z_i)$ uses actual covariate values (not Z.ref).

**AME (continuous)**:

$$AME = \frac{\sum_i w_i \cdot ME(X_i, D_i, Z_i)}{\sum_i w_i}$$

ATE/AME delta-method SE: average the gradient vectors across observations, then $SE = \sqrt{\bar{g}' V \bar{g}}$.

### vcovCluster

$$\hat{V}_{cluster} = \frac{M}{M-1} \cdot \frac{N-1}{N-K} \cdot \text{sandwich}(\hat\beta, B, \frac{1}{N} \sum_c u_c' u_c)$$

where $u_c = \sum_{i \in c} s_i$ (sum of score contributions within cluster $c$), $B$ = bread matrix, $M$ = number of clusters, $N$ = observations, $K$ = model rank.

### FWL Demeaning (fastplm.cpp)

Iterative algorithm:
1. For each FE variable in turn, compute weighted group means and subtract from Y and all X columns
2. Repeat until $\sum |data_{new} - data_{old}| < 10^{-5}$ or 50 iterations
3. Apply $\sqrt{w}$ weighting to demeaned data
4. Run OLS: $\hat\beta = (X'X)^{-1} X'y$ (via `arma::solve`)
5. Residuals: $(y - X\hat\beta) / \sqrt{w}$
6. Grand mean: $\mu = \bar{y}_w - \bar{X}_w' \hat\beta$

### IV FWL (iv_fastplm.cpp)

Same demeaning step, then 2SLS:
1. $Z_{full} = [Z_{exog}, IV]$, $X_{full} = [X_{endog}, Z_{exog}]$
2. $P_Z = Z_{full} (Z_{full}' Z_{full})^{-1} Z_{full}'$
3. $\hat\beta = (P_Z X_{full})^{-1} (P_Z Y) = \text{solve}(P_Z X_{full}, P_Z Y)$

### Uniform Confidence Intervals

**Bootstrap uniform CI** (`calculate_uniform_quantiles`):
- Input: matrix of bootstrap TE/ME draws (k evaluation points x N bootstrap draws)
- Binary search for $\zeta$ in $[\alpha/(2k), \alpha/2]$ such that the fraction of bootstrap draws where ALL k evaluation points fall within their pointwise $[\zeta, 1-\zeta]$ quantile bands is $\geq 1-\alpha$.
- Uses Bonferroni $\zeta = \alpha/(2k)$ as fallback.

**Delta uniform CI** (`calculate_delta_uniformCI`):
- Draw $V \sim \mathcal{N}(0, \Sigma)$ where $\Sigma$ is the TE/ME covariance matrix across evaluation points
- Compute $\max_j |V_j / \sqrt{\Sigma_{jj}}|$ for each draw
- $q = (1-\alpha)$-quantile of the max statistic
- Uniform CI: $\hat\theta(x) \pm q \cdot SE(x)$

### Differences

For 2 evaluation points $(x_1, x_2)$: diff = $TE(x_2) - TE(x_1)$
For 3 points $(x_1, x_2, x_3)$: diffs = $(TE(x_2)-TE(x_1), TE(x_3)-TE(x_2), TE(x_3)-TE(x_1))$

Delta SE for differences: $SE_{diff} = \sqrt{(g_2 - g_1)' V (g_2 - g_1)}$

## Questions and Answers
None needed.

## Comprehension Verdict
**FULLY UNDERSTOOD**

All symbols defined. All formulas verified against source code. No ambiguous steps. The translation is mechanical: the mathematical content is fixed by the R code, and the Python equivalents are well-defined.

## Implicit Assumptions Noted

1. `method = "linear"` is the only method compatible with FE and IV. The R code enforces this in `interflex.R`.
2. PCSE requires both `cl` and `time` specified; it is not compatible with FE (auto-switches to cluster).
3. Negative binomial uses `MASS::glm.nb` which estimates the dispersion parameter theta via MLE. In Python, `statsmodels.discrete.models.NegativeBinomial` uses a different parameterization (NB2 by default) — the builder must handle this parameterization difference.
4. The FWL demeaning convergence tolerance (1e-5) and max iterations (50) are hardcoded in C++ and should be replicated exactly.
5. For FE models, predicted values and link SDs are returned as 0 — this is by design since FE absorbs the level.
6. The `full.moderate` option adds Z*X interactions to the linear predictor, expanding both the formula and the gradient vectors.
7. Bootstrap is block bootstrap when `cl` is specified.
8. The simulation variance uses `rmvnorm()` which draws from MVN(coef, vcov) — this is a parametric approach, not a bootstrap.
