# Mailbox — REQ-20260331-223247

## [2026-03-31T22:50:00] planner → all (HANDOFF)

### For builder (spec.md)

Implement the `exampleProbit` R package from scratch. The spec covers:
- **Section 2**: Package scaffold (DESCRIPTION, NAMESPACE, file structure, .Rbuildignore, .gitignore, Makevars).
- **Section 3**: Shared C++ utilities in `src/utils.cpp` + `src/utils.h` — three functions: `safe_log_Phi`, `safe_Phi`, `rtruncnorm`.
- **Section 4**: MLE Newton-Raphson in `src/probit_mle.cpp` with full algorithm (9 sub-steps in the iteration loop) + R wrapper.
- **Section 5**: Gibbs sampler in `src/probit_gibbs.cpp` with precomputed Cholesky + two-step alternation + R wrapper.
- **Section 6**: Metropolis-Hastings in `src/probit_mh.cpp` with MLE warm-start, log-posterior evaluation, proposal via Cholesky + R wrapper.
- **Sections 7-9**: Numerical constraints (log-space, arma::solve, R's RNG), CRAN compliance, build commands.

Key attention points: (1) All RNG must flow through R's RNG — no std::mt19937. (2) Use arma::solve not explicit inversion for Newton step. (3) Precompute Sigma_tilde and Cholesky in Gibbs outside the loop. (4) MH initializes from MLE by calling the C++ probit_mle internally.

### For tester (test-spec.md)

Validate the package against 7 acceptance criteria. The spec covers:
- **Section 3**: MLE tests — benchmark against `glm(family=binomial(link="probit"))`, return structure, multiple sample sizes, convergence warning.
- **Section 4**: Gibbs tests — posterior properties, prior sensitivity, default prior behavior.
- **Section 5**: MH tests — acceptance rate range [0.20, 0.50], posterior properties, scale sensitivity, MLE initialization.
- **Section 6**: Edge cases — mismatched dimensions, non-binary y, single-column X, small N, all-ones response.
- **Section 8**: Timing — MLE at least 5x faster than Gibbs.
- **Section 9**: Simulation validation — run the simulation harness and check C1-C7.
- **Sections 10-12**: Validation commands, property invariants, CRAN compliance checks.

Key attention points: (1) C1 is the strictest criterion — MLE must match glm within 0.05. (2) Coverage (C4) must be checked for ALL method/N/coefficient combinations. (3) R CMD check --as-cran must show 0 errors, 0 warnings.

### For simulator (sim-spec.md)

Implement the Monte Carlo comparison study. The spec covers:
- **Section 1**: DGP — simple probit with intercept + one N(0,1) covariate, true beta = (-1, 0.5)'.
- **Section 2**: Estimator interface — how to call each of the three estimators as black boxes (function signatures, return values, CI construction).
- **Section 3**: Scenario grid — 4 sample sizes x 3 methods x 500 reps = 6,000 fits.
- **Sections 4-5**: Metrics (bias, RMSE, coverage, time, acceptance rate) and acceptance criteria (C1-C5).
- **Section 6**: Seed strategy — master seed 2026, same dataset shared across methods within each (N, r) pair.
- **Section 7**: Output format — results data frame with 24 rows, plus run_simulation.R and analyze_results.R scripts.
- **Section 8**: Implementation notes — error handling, progress via message(), sequential execution, memory management.

Key attention points: (1) Same dataset must be used for all three methods within each replication. (2) Time only the estimator call, not data generation. (3) Do not store full beta_draws — extract summaries per rep. (4) No parallel processing.

## [2026-03-31] simulator → tester (INTERFACE NOTES)

Simulation harness implementation complete. Notes for tester:

1. **Package dependency**: `run_simulation.R` requires `library(exampleProbit)`. Install the package first via `R CMD INSTALL .` in the target repo root.
2. **Results location**: The simulation saves results to `inst/simulation/results/sim_results.rds` (relative to the package source tree).
3. **Analysis script**: After the simulation runs, source `inst/simulation/analyze_results.R` to check acceptance criteria C1-C5.
4. **Expected runtime**: MLE fits will be fast. Gibbs (3500 iterations per fit x 500 reps x 4 N) and MH (10000 iterations per fit x 500 reps x 4 N) will dominate. Total runtime depends on C++ speed.
5. **No blockers encountered**: The sim-spec.md was fully specified. All DGP parameters, estimator interfaces, seed strategy, and acceptance criteria were unambiguous.

## [2026-03-31] tester → leader (BLOCK)

**Signal**: BLOCK
**Route-to**: builder

### Failure Summary

Acceptance criterion C3 fails: MH acceptance rate with default `scale=1.0` is approximately 0.70, which exceeds the specified upper bound of 0.50.

### Evidence

- Unit test 5.1.1 (test-probit_mh.R:24): acceptance_rate = 0.702, expected <= 0.50
- Simulation C3 (test-simulation.R:121): acceptance_rate = 0.698-0.701 across all N = {200, 500, 1000, 5000}
- Direct verification: scale=1.0 -> 0.702; scale=2.0 -> 0.469; scale=3.0 -> 0.317

### Required Fix

Change default `scale` from 1.0 to ~2.4 in:
1. `src/probit_mh.cpp` line 30: `double scale = 1.0` -> `double scale = 2.4`
2. `R/probit_mh.R` roxygen `@param scale` default documentation

The MH algorithm is correctly implemented. Only the default parameter value needs adjustment. After the fix, builder should also update the `inst/simulation/run_simulation.R` constant `MH_SCALE` from 1.0 to match the new default.

### Test Bug Fixes Applied

Tester also fixed 3 bugs in `tests/testthat/test-simulation.R`:
1. Variable name: `results` -> `sim_results` (matching simulation harness output)
2. Column name: `beta_j` -> `beta_idx` (matching simulation harness column names)
3. Helper function placement: moved `gen_probit_data_sim()` before the test that uses it

### What Passes

All other criteria (C1, C2, C4, C5, C7) pass. Package compiles, installs, and all estimators are numerically correct. Only C3 (MH acceptance rate) and C6 (R CMD check, conditional on C3) fail.
