# Simulation Artifact: Probit Monte Carlo Study

```yaml
RequestID: REQ-20260331-223247
Agent: simulator
Status: SIMULATED
```

---

## 1. Simulation Design Summary

### 1.1 DGP

Binary probit model: Pr(y_i = 1 | x_i) = Phi(x_i' beta_0)

- True parameters: beta_0 = (-1, 0.5)' (intercept, slope)
- Covariate: x1 ~ N(0, 1), design matrix X = (1, x1)
- Response: y ~ Bernoulli(Phi(-1 + 0.5 * x1))

### 1.2 Estimator Interface

All three estimators are called as black boxes from the `exampleProbit` package:

| Method | Call | Point Estimate | CI Construction |
|--------|------|----------------|-----------------|
| MLE | `probit_mle(X, y, max_iter=100, tol=1e-8)` | `coefficients` | beta_hat +/- 1.96 * SE |
| Gibbs | `probit_gibbs(X, y, n_iter=3500, burn_in=500, beta0=rep(0,2), Sigma0=diag(100,2))` | `posterior_mean` | 2.5%/97.5% quantiles of `beta_draws` |
| MH | `probit_mh(X, y, n_iter=10000, burn_in=2000, scale=1.0, beta0=rep(0,2), Sigma0=diag(100,2))` | `posterior_mean` | 2.5%/97.5% quantiles of `beta_draws` |

### 1.3 Scenario Grid

| Dimension | Values | Count |
|-----------|--------|-------|
| Sample size N | 200, 500, 1000, 5000 | 4 |
| Method | MLE, Gibbs, MH | 3 |

Total scenarios: 12. With R = 500 replications each: 6,000 estimator fits.

### 1.4 Performance Metrics

Computed per method, per coefficient (intercept, slope), across R = 500 replications:

- **Bias**: mean(beta_hat - beta_true)
- **RMSE**: sqrt(mean((beta_hat - beta_true)^2))
- **Coverage**: proportion of 95% CIs containing beta_true
- **Mean Time**: mean wall-clock seconds per estimator call
- **Acceptance Rate**: mean MH acceptance rate (MH only)
- **Failure Rate**: proportion of replications that errored

### 1.5 Seed Strategy

- Master seed: 2026
- Per sample-size block: `set.seed(2026 + N)` generates R = 500 shared datasets
- Per method sub-seeds: `set.seed(2026*100 + N*10 + r + offset)` where offset = 0 (MLE), 10000 (Gibbs), 20000 (MH)
- Same dataset (X, y) is used by all three methods within each (N, r) pair
- Sequential execution (no parallelism) for full reproducibility

---

## 2. Code Summary

### 2.1 Files Created

| File | Description |
|------|-------------|
| `inst/simulation/run_simulation.R` | Main simulation driver. Generates data, runs all 6,000 fits, computes metrics, saves `sim_results.rds`, prints summary table. |
| `inst/simulation/analyze_results.R` | Analysis script. Loads `sim_results.rds`, produces formatted tables, evaluates acceptance criteria C1-C5. |

### 2.2 Simulation Harness Structure

1. **Data pre-generation**: For each N, set seed and generate all R = 500 datasets upfront. Stored in a list, shared across methods.
2. **Method loop**: For each method, iterate over the R pre-generated datasets. Set a per-replication sub-seed before each estimator call.
3. **Error handling**: Each estimator call is wrapped in `tryCatch()`. Failures are recorded as NA with `success = FALSE`. Failure rate is reported per scenario.
4. **Timing**: Only the estimator call is timed via `proc.time()`.
5. **Memory management**: Point estimates, CIs, and timing are extracted immediately; full `beta_draws` matrices are discarded after extracting quantiles.
6. **Progress reporting**: `message()` every 100 replications.

### 2.3 Output Structure

Results data frame: 24 rows (4 N x 3 methods x 2 coefficients), columns: method, N, beta_idx, coef_label, bias, rmse, coverage, mean_time, acceptance_rate, n_success, n_fail, failure_rate.

Saved as `inst/simulation/results/sim_results.rds`.

---

## 3. Smoke Run

Smoke run was NOT executed because the `exampleProbit` package is not yet installed (builder is implementing it in parallel). The simulation harness is syntactically valid R code and was reviewed for correctness against the sim-spec.md specification.

The tester will execute the full simulation after the package is built and installed.

---

## 4. Acceptance Criteria Assessment

The acceptance criteria cannot be numerically evaluated until the simulation is run. The `analyze_results.R` script implements all five checks:

| ID | Criterion | Implementation |
|----|-----------|----------------|
| C1 | MLE matches glm (bias < 0.05) | Checks max absolute MLE bias across all (N, coef) |
| C2 | Bayesian-Frequentist agreement | Compares bias difference between MLE and Gibbs/MH for N >= 500 |
| C3 | MH acceptance rate in [0.20, 0.50] | Checks all N values |
| C4 | 95% CI coverage in [0.90, 0.99] | Checks all (method, N, coef) combinations |
| C5 | MLE speed advantage (< Gibbs/5) | Compares mean times for each N |

---

## 5. Diagnostic Notes

- **Data sharing**: Datasets are pre-generated and shared across methods for fair comparison. This is critical: without shared data, differences between methods could be due to different random draws rather than estimator properties.
- **Sub-seed strategy**: Each (method, N, r) triple gets a unique seed offset. This ensures MCMC randomness is reproducible independently of other methods. MLE is deterministic given data, but gets a sub-seed for consistency.
- **Memory**: The harness discards `beta_draws` after extracting quantiles to avoid storing up to 500 * 8000 * 2 = 8M doubles for MH alone.
- **No parallelism**: Sequential execution per sim-spec.md section 8.6. This avoids CRAN compliance issues and ensures full reproducibility.
- **Failure tracking**: All three methods report failure rates. A rate > 5% should be flagged by the tester.

---

## 6. Verdict

**SIMULATED** — Simulation harness code written. Both `run_simulation.R` and `analyze_results.R` are complete and follow the sim-spec.md specification exactly. The code cannot be smoke-tested until builder completes the `exampleProbit` package. Tester will execute the full simulation and verify acceptance criteria.

---

## 7. Interface Notes for Tester

- The simulation script requires `library(exampleProbit)` — package must be installed first.
- Results are saved to `inst/simulation/results/sim_results.rds`.
- Run `source("inst/simulation/analyze_results.R")` after the simulation to check criteria.
- Expected runtime: depends on C++ estimator speed. MLE should be fast; Gibbs and MH will dominate. For N=5000 with R=500, the MH loop (10,000 iterations each) may take significant time.
