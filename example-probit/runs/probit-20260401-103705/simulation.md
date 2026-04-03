# Simulation Artifact (simulation.md)

```yaml
Request ID: probit-20260401-103705
Pipeline: Simulation
Agent: simulator
Verdict: SIMULATED
```

---

## 1. Simulation Design Summary

### DGP

Binary probit model with one covariate:

    Pr(y = 1 | x_1) = Phi(-1 + 0.5 * x_1)

Latent variable formulation:

    y* = beta_0 + beta_1 * x_1 + eps,  eps ~ N(0, 1)
    y = 1(y* > 0)

- True parameters: beta = (-1, 0.5)'
- Covariate: x_1 ~ N(0, 1) iid
- Design matrix: X = cbind(1, x_1) is N x 2

### Scenario Grid

| Dimension | Values | Count |
| --- | --- | --- |
| Sample size N | {200, 500, 1000, 5000} | 4 |
| Method | {MLE, Gibbs, MH} | 3 |
| Parameters | {beta_0, beta_1} | 2 |

Total scenarios: 4 x 3 = 12 (24 rows in summary table with 2 parameters each)

### Replications

R = 500 per scenario. Total estimator fits: 6000.

### Seed Strategy

- Master seed: 2026
- Per-replication seeds pre-generated via `set.seed(2026); seeds <- sample.int(.Machine$integer.max, 500)`
- For each replication r: `set.seed(seeds[r])` before generating data
- Same (X, y) data used for all three methods within the same (N, r) combination

### Estimator Configurations

| Method | Parameter | Value |
| --- | --- | --- |
| MLE | max_iter | 100 |
| MLE | tol | 1e-8 |
| Gibbs | n_iter | 3500 |
| Gibbs | burn_in | 500 |
| MH | n_iter | 10000 |
| MH | burn_in | 2000 |
| MH | scale | 1.0 |
| Gibbs & MH | beta0 (prior mean) | rep(0, 2) |
| Gibbs & MH | Sigma0 (prior cov) | diag(100, 2) |

---

## 2. Code Summary

### Files Created

| File | Purpose |
| --- | --- |
| `inst/simulation/run_simulation.R` | Standalone simulation script (run via `Rscript`) |
| `R/simulate_probit.R` | Exported R function `run_probit_simulation()` + internal helper `.compute_sim_metrics()` |

### `inst/simulation/run_simulation.R`

Main simulation script structure:
1. Loads `exampleProbit` package
2. Sets configuration (beta_true, N_vec, R, master_seed, estimator args)
3. Pre-generates R seeds from master seed
4. Outer loop over sample sizes N in {200, 500, 1000, 5000}
5. Inner loop over replications r in 1:500
   - Sets seed, generates (X, y) once per replication
   - Calls `probit_mle()`, `probit_gibbs()`, `probit_mh()` on the same data
   - Records point estimates, CI bounds, timing, convergence/acceptance
   - Each estimator call wrapped in `tryCatch()` for error handling
6. Computes summary metrics (bias, RMSE, coverage, time, acceptance rate)
7. Prints formatted results table
8. Saves raw + summary results to RDS file

### `R/simulate_probit.R`

- `run_probit_simulation(N_vec, R, seed, verbose)`: Exported function wrapping the simulation. Returns list with `raw_results` and `summary_table`. Defaults match sim-spec.md. Uses `stats::` namespace qualification for CRAN compliance.
- `.compute_sim_metrics()`: Internal helper that computes bias, RMSE, coverage, time, and acceptance rate for one (N, method, parameter) combination.
- roxygen2 documentation with `\donttest{}` example (computationally expensive).

### CI Construction

- MLE: `beta_hat +/- 1.96 * SE` (asymptotic normality)
- Gibbs: 2.5% and 97.5% quantiles of `beta_draws` columns
- MH: 2.5% and 97.5% quantiles of `beta_draws` columns

### Error Handling

Each estimator call is wrapped in `tryCatch()`. Failures are:
- Logged via `message()` with N, r, and error text
- Recorded as `NA` in raw results
- Counted in `n_fail` column of summary table
- Excluded from metric computation (metrics computed over valid replications only)

### Parallelization

None. Sequential execution per sim-spec.md Section 9 (CRAN compliance: max 2 cores in tests/examples). The script runs all 6000 fits sequentially.

---

## 3. Smoke Run Results

Smoke run was NOT executed because the package (builder's output) is being built in parallel and is not yet installable. The simulation code is syntactically valid R and follows the estimator interfaces documented in sim-spec.md exactly. Tester will execute the full simulation as part of validation.

---

## 4. Acceptance Criteria Assessment

The simulation code is designed to produce data that allows evaluation of all acceptance criteria:

| ID | Criterion | How Measured |
| --- | --- | --- |
| C1 | MLE matches glm within tolerance | Tester will compare `probit_mle()` vs `glm()` per-replication. Not directly in simulation script (this is a unit test concern). |
| C2 | Gibbs/MH posterior means within 0.1 of MLE bias (N >= 500) | Summary table bias columns can be compared across methods for N >= 500. |
| C3 | MH acceptance rate in [0.20, 0.50] | `accept_rate` column in summary table. |
| C4 | 95% CI coverage in [0.90, 0.99] | `coverage` column in summary table. |
| C5 | MLE at least 5x faster than Gibbs | `time` column ratios in summary table. |

Criteria can only be fully evaluated after the simulation runs (tester's job).

---

## 5. Diagnostic Notes

- The simulation script treats estimators as black boxes per pipeline isolation rules. It calls `probit_mle()`, `probit_gibbs()`, and `probit_mh()` exactly as documented in sim-spec.md Section 2.
- Memory: raw results stored as a list of lists. For R=500 and 4 sample sizes, this is 2000 list elements. Each contains small numeric vectors. Memory usage should be modest (< 100 MB).
- Timing: `proc.time()` elapsed time is used (wall-clock). This captures the full estimator call including R-to-C++ overhead.
- MH at N=5000 with 10000 iterations will be the most expensive call. Estimated 0.5-2s per fit, so ~500-1000s for that cell alone.
- The RDS output path logic handles both installed-package and source-tree contexts.
- Progress messages every 100 replications to indicate the simulation is running.

---

## 6. Verdict

**SIMULATED** -- Simulation code written. Both files are syntactically valid R that follows the exact estimator interfaces from sim-spec.md. Smoke run deferred to tester (package not yet installable while builder runs in parallel with simulator). All acceptance criteria are measurable from the output.
