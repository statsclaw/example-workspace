# Mailbox

## Handoff Summary (from planner)

**Timestamp**: 2026-04-01

### For Builder (spec.md)

Builder must implement the `exampleProbit` R package from scratch. The spec covers: package structure (DESCRIPTION, NAMESPACE via roxygen2, Makevars), three C++ source files (`probit_mle.cpp`, `probit_gibbs.cpp`, `probit_mh.cpp`) with exact algorithm steps and Armadillo matrix operations, shared utilities in `utils.cpp` (safe_log_Phi, safe_Phi, rtruncnorm), and R wrapper functions with roxygen2 documentation in `R/`. Key implementation details: MLE uses Newton-Raphson with solve (not explicit inversion), Gibbs precomputes Sigma_tilde and its Cholesky outside the loop, MH initializes from MLE and uses log-scale acceptance. All numerical stability requirements are in spec.md Section 4.3, 5.3, and 6.3. CRAN compliance rules are in Section 8. Name C++ exports with `_cpp` suffix to avoid R wrapper collision.

### For Tester (test-spec.md)

Tester must validate the package via R CMD check (C6), R CMD INSTALL (C7), and a testthat suite covering all 7 acceptance criteria. Unit tests T1-T17 cover output structure, glm agreement (C1), Bayesian-MLE consistency (C2), MH acceptance rate (C3), and large-sample accuracy. Edge cases E1-E5 cover small N, degenerate outcomes, intercept-only, zero burn-in, and high-dimensional designs. Property tests P1-P4 verify log-likelihood monotonicity, posterior concentration, scale-acceptance tradeoff, and coefficient symmetry. Simulation validation section (6.1-6.2) specifies how to check C1-C5 from the simulator's output. All tolerances are in Section 10 -- they must never be relaxed.

### For Simulator (sim-spec.md)

Simulator must implement the Monte Carlo simulation study with DGP Phi(-1 + 0.5*x1), x1 ~ N(0,1) across N = {200, 500, 1000, 5000} with R = 500 replications. The script goes in `inst/simulation/run_simulation.R`. For each (N, r), generate data once and run all three estimators on the same data. Compute Bias, RMSE, Coverage, and Time per method/coefficient/N. Save raw results as RDS and summary table in simulation.md. Seed strategy uses master seed 2026 with pre-generated per-replication seeds. CI construction differs by method (asymptotic for MLE, quantile for Bayesian). Do not parallelize. Report progress via message().

## Simulator Notes (from simulator)

**Timestamp**: 2026-04-01

### Files Created

1. `inst/simulation/run_simulation.R` -- Standalone simulation script. Run via `Rscript inst/simulation/run_simulation.R` after installing the package. Saves raw results to `inst/simulation/simulation_results.rds`.
2. `R/simulate_probit.R` -- Exported function `run_probit_simulation()` with roxygen2 docs. Internal helper `.compute_sim_metrics()`.

### Interface Notes for Tester

- The simulation calls `probit_mle(X, y, max_iter, tol)`, `probit_gibbs(X, y, n_iter, burn_in, beta0, Sigma0)`, and `probit_mh(X, y, n_iter, burn_in, scale, beta0, Sigma0)` exactly as specified in sim-spec.md Section 2.
- If builder changes the function signatures, the simulation script will fail with clear errors from `tryCatch()`.
- The `run_probit_simulation()` function in `R/simulate_probit.R` can be used programmatically for testing with reduced R (e.g., R=10).
- Smoke run was deferred because the package is being built by builder in parallel with simulator. Tester should install the package first, then run the simulation.
- Raw results RDS file includes both raw per-replication data and the summary table, plus all configuration parameters for audit.

## BLOCK Signal (from tester)

**Timestamp**: 2026-04-01

### Signal: BLOCK

**Failed Criterion**: C3 -- MH acceptance rate must be in [0.20, 0.50]

**Observed**: MH acceptance rate is approximately 0.70 across all sample sizes (N=200: 0.700, N=500: 0.699, N=1000: 0.698, N=5000: 0.697). This is consistently above the upper bound of 0.50.

**Root Cause**: The proposal covariance in `src/probit_mh.cpp` is `scale^2 * (X'X)^{-1}` with `scale = 1.0`. This produces proposals that are too close to the current state, leading to an excessively high acceptance rate. The acceptance rate is insensitive to N, suggesting the scale-to-covariance relationship is structurally too conservative.

**Route To**: **builder**

**Required Fix**: Increase the default `scale` parameter in `probit_mh()` (both `R/probit_mh.R` and `src/probit_mh.cpp` defaults) so that with `scale = 1.0` or the new default, the acceptance rate falls within [0.20, 0.50]. Alternatively, change the proposal covariance formula. After builder fixes, the simulation configurations in `R/simulate_probit.R` (line 59) and `inst/simulation/run_simulation.R` (line 31) may also need their `scale` parameter updated.

**All Other Criteria PASS**: C1, C2, C4, C5, C6, C7 all pass. Only C3 fails.
