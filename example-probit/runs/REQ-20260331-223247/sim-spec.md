# Simulation Specification: Probit Monte Carlo Study

```yaml
RequestID: REQ-20260331-223247
Pipeline: Simulation (simulator only)
Profile: r-package
```

---

## 1. DGP Definition

### 1.1 Model

Binary probit model: Pr(y_i = 1 | x_i) = Phi(x'_i * beta_0), where Phi is the standard normal CDF.

### 1.2 True Parameters

| Parameter | Value | Type |
|-----------|-------|------|
| beta_0 | (-1, 0.5)' | k x 1 vector, k = 2 |
| beta_0[1] | -1 | intercept |
| beta_0[2] | 0.5 | slope on x1 |

### 1.3 Covariate Distribution

- x1_i ~ N(0, 1) independently for i = 1, ..., N.
- Design matrix: X_i = (1, x1_i)' (intercept + one covariate).
- Full design matrix: X is N x 2.

### 1.4 Response Generation

For each observation i:
1. Draw x1_i ~ N(0, 1).
2. Compute prob_i = Phi(-1 + 0.5 * x1_i).
3. Draw y_i ~ Bernoulli(prob_i).

### 1.5 Sample Sizes

N in {200, 500, 1000, 5000}.

---

## 2. Estimator Interface (Black Box)

Simulator treats each estimator as a black box. Call via R functions from the `exampleProbit` package.

### 2.1 MLE

**Call**: `probit_mle(X, y, max_iter = 100, tol = 1e-8)`

**Returns**: A list with:
- `coefficients`: numeric vector of length k (point estimates)
- `se`: numeric vector of length k (standard errors)
- `vcov`: numeric matrix k x k (variance-covariance)
- `loglik`: scalar (log-likelihood at MLE)

**CI construction**: For coefficient j, CI = [coefficients[j] - 1.96 * se[j], coefficients[j] + 1.96 * se[j]].

### 2.2 Gibbs

**Call**: `probit_gibbs(X, y, n_iter = 3500, burn_in = 500, beta0 = rep(0, 2), Sigma0 = diag(100, 2))`

**Returns**: A list with:
- `beta_draws`: matrix (n_iter - burn_in) x k (posterior draws after burn-in)
- `posterior_mean`: numeric vector of length k
- `posterior_sd`: numeric vector of length k

**Point estimate**: `posterior_mean`.
**CI construction**: For coefficient j, CI = [quantile(beta_draws[,j], 0.025), quantile(beta_draws[,j], 0.975)].

### 2.3 MH

**Call**: `probit_mh(X, y, n_iter = 10000, burn_in = 2000, scale = 1.0, beta0 = rep(0, 2), Sigma0 = diag(100, 2))`

**Returns**: A list with:
- `beta_draws`: matrix (n_iter - burn_in) x k
- `posterior_mean`: numeric vector of length k
- `posterior_sd`: numeric vector of length k
- `acceptance_rate`: scalar in [0, 1]

**Point estimate**: `posterior_mean`.
**CI construction**: Same as Gibbs (2.5% and 97.5% quantiles of beta_draws).

---

## 3. Scenario Grid

### 3.1 Dimensions

| Dimension | Values | Count |
|-----------|--------|-------|
| Sample size N | 200, 500, 1000, 5000 | 4 |
| Estimation method | MLE, Gibbs, MH | 3 |

Total scenarios: 4 x 3 = 12.

### 3.2 Replications

R = 500 per scenario.

### 3.3 Total Estimation Runs

12 scenarios x 500 replications = 6,000 fits.

### 3.4 Estimator Settings per Method

| Method | Setting | Value |
|--------|---------|-------|
| MLE | max_iter | 100 |
| MLE | tol | 1e-8 |
| Gibbs | n_iter | 3500 |
| Gibbs | burn_in | 500 |
| Gibbs | beta0 | rep(0, 2) |
| Gibbs | Sigma0 | diag(100, 2) |
| MH | n_iter | 10000 |
| MH | burn_in | 2000 |
| MH | scale | 1.0 |
| MH | beta0 | rep(0, 2) |
| MH | Sigma0 | diag(100, 2) |

---

## 4. Performance Metrics

All metrics are computed **per method**, **per coefficient j** (j = 1 for intercept, j = 2 for slope), **across R = 500 replications**.

### 4.1 Bias

```
Bias_j = (1/R) * sum_{r=1}^{R} (beta_hat_j^{(r)} - beta_{0,j})
```

Where beta_hat_j^{(r)} is the point estimate from replication r.
- For MLE: beta_hat = coefficients.
- For Gibbs/MH: beta_hat = posterior_mean.

### 4.2 RMSE

```
RMSE_j = sqrt( (1/R) * sum_{r=1}^{R} (beta_hat_j^{(r)} - beta_{0,j})^2 )
```

### 4.3 Coverage

```
Coverage_j = (1/R) * sum_{r=1}^{R} 1(beta_{0,j} in CI_j^{(r)})
```

Where CI_j^{(r)} is the 95% confidence/credible interval from replication r.

CI construction:
- MLE: [beta_hat_j - 1.96 * SE_j, beta_hat_j + 1.96 * SE_j]
- Gibbs/MH: [quantile(beta_draws[,j], 0.025), quantile(beta_draws[,j], 0.975)]

### 4.4 Mean Computation Time

```
Time = (1/R) * sum_{r=1}^{R} t^{(r)}
```

Where t^{(r)} is the wall-clock time in seconds for replication r. Measure using `system.time()` or `proc.time()`. Include only the estimation call, not data generation.

### 4.5 Acceptance Rate (MH only)

```
MeanAcceptRate = (1/R) * sum_{r=1}^{R} acceptance_rate^{(r)}
```

---

## 5. Acceptance Criteria

These are the thresholds that define whether the simulation results are acceptable.

| ID | Criterion | Threshold |
|----|-----------|-----------|
| C1 | MLE matches glm | Mean abs difference < 0.05 for all N |
| C2 | Bayesian-Frequentist agreement | Gibbs/MH posterior mean within 0.1 of MLE for N >= 500 |
| C3 | MH acceptance rate | Mean acceptance rate in [0.20, 0.50] for all N |
| C4 | 95% CI coverage | Coverage in [0.90, 0.99] for all methods, all N, both coefficients |
| C5 | MLE speed advantage | MLE mean time < Gibbs mean time / 5 for all N |

Note: C6 and C7 are package-level criteria (R CMD check, installability) — not simulation metrics. They are verified by tester separately.

---

## 6. Seed Strategy

### 6.1 Master Seed

Master seed: 2026.

### 6.2 Per-Scenario Seed Derivation

For each scenario (method m, sample size N), the per-replication seeds are derived as:

```r
set.seed(2026)
# Generate all data first (shared across methods for each N)
```

**Important**: For each sample size N and each replication r, the **same dataset** (X, y) must be used across all three methods. This ensures fair comparison.

Implementation approach:
1. For each N:
   a. Set `set.seed(2026 + N)` at the start of this sample-size block.
   b. Pre-generate R = 500 datasets: for each r, draw X and y.
   c. For each method, fit on each of the 500 pre-generated datasets.

This ensures:
- The same data is used for MLE, Gibbs, and MH within each (N, r) pair.
- Results are reproducible.
- Different sample sizes use different seed sequences.

### 6.3 RNG Type

Use R's default RNG (Mersenne-Twister). Do not call `RNGkind()` — use whatever is set.

**MCMC note**: Gibbs and MH use R's RNG internally (through RcppArmadillo's arma::randn and R::runif). Each method call advances the RNG state. To ensure reproducibility within a replication, set a per-replication sub-seed before each method call:

```r
for (r in 1:R) {
  # Generate data with the pre-set seed sequence
  ...
  # MLE (deterministic given data, no RNG needed beyond data gen)
  set.seed(2026 * 100 + N * 10 + r)  # sub-seed for MLE (not strictly needed)
  mle_result <- probit_mle(X, y)

  set.seed(2026 * 100 + N * 10 + r + 10000)  # sub-seed for Gibbs
  gibbs_result <- probit_gibbs(X, y, ...)

  set.seed(2026 * 100 + N * 10 + r + 20000)  # sub-seed for MH
  mh_result <- probit_mh(X, y, ...)
}
```

The exact sub-seed formula is flexible — the key requirement is that each (method, N, r) triple gets a unique, reproducible seed.

---

## 7. Output Format

### 7.1 Results Data Frame

The simulation must produce a data frame with the following columns:

| Column | Type | Description |
|--------|------|-------------|
| method | character | "MLE", "Gibbs", or "MH" |
| N | integer | Sample size |
| beta_idx | integer | 1 (intercept) or 2 (slope) |
| bias | numeric | Bias for this (method, N, beta) |
| rmse | numeric | RMSE for this (method, N, beta) |
| coverage | numeric | Empirical 95% CI coverage |
| mean_time | numeric | Mean computation time (seconds) |
| acceptance_rate | numeric | Mean acceptance rate (NA for MLE and Gibbs) |

This data frame has 4 (N) x 3 (methods) x 2 (coefficients) = 24 rows.

### 7.2 Output Files

Simulator must produce:

1. **`inst/simulation/run_simulation.R`** — The executable simulation script. When sourced, it:
   - Loads the `exampleProbit` package.
   - Runs all 6,000 fits.
   - Computes all metrics.
   - Saves the results data frame to `inst/simulation/sim_results.rds`.
   - Prints a summary table to console.

2. **`inst/simulation/analyze_results.R`** — Analysis script that:
   - Loads `sim_results.rds`.
   - Produces formatted tables (bias, RMSE, coverage, time) by method and N.
   - Checks each acceptance criterion (C1-C5) and prints PASS/FAIL.

3. **Simulator writes `simulation.md`** to the run directory with:
   - Summary of implementation decisions.
   - The results table.
   - PASS/FAIL status for each criterion.

### 7.3 Summary Table Format

The summary table should look like:

```
Method | N    | Coef      | Bias    | RMSE   | Coverage | Time (s)  | Accept Rate
-------|------|-----------|---------|--------|----------|-----------|------------
MLE    | 200  | intercept | ...     | ...    | ...      | ...       | NA
MLE    | 200  | slope     | ...     | ...    | ...      | ...       | NA
...
Gibbs  | 200  | intercept | ...     | ...    | ...      | ...       | NA
...
MH     | 200  | intercept | ...     | ...    | ...      | ...       | ...
...
```

---

## 8. Implementation Notes for Simulator

### 8.1 File Locations

- Simulation scripts go in `inst/simulation/` in the target repo.
- The `simulation.md` artifact goes in the run directory.

### 8.2 Error Handling

- If any single fit fails (e.g., MLE non-convergence), catch the error, record NA for that replication, and continue. Report the failure rate.
- A failure rate > 5% for any (method, N) combination should be flagged.

### 8.3 Progress Reporting

- Use `message()` to report progress (e.g., "N=200, replication 100/500").
- Do not use `cat()` or `print()` for progress — use `message()` for CRAN compliance.

### 8.4 Timing

- Time only the estimation call, not data generation or metric computation.
- Use `proc.time()` around each estimator call:
  ```r
  t0 <- proc.time()
  result <- probit_mle(X, y)
  elapsed <- (proc.time() - t0)[3]
  ```

### 8.5 Memory

- Do not store all 6,000 full beta_draws matrices. Extract point estimates, CIs, and timing per replication, then discard the draws.
- Pre-allocate result storage matrices before the replication loop.

### 8.6 Parallelism

- Do NOT use parallel processing in the simulation script. Run sequentially.
- This ensures reproducibility and avoids CRAN compliance issues (max 2 cores rule).
