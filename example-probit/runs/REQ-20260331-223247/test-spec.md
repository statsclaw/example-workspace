# Test Specification: exampleProbit

```yaml
RequestID: REQ-20260331-223247
Pipeline: Test (tester only)
Profile: r-package
```

---

## 1. Behavioral Contract

The `exampleProbit` package provides three probit estimation functions:

1. **`probit_mle(X, y, max_iter, tol)`** — Returns a list with `coefficients` (numeric vector), `vcov` (numeric matrix), `se` (numeric vector), and `loglik` (scalar). The coefficients must match R's `glm(..., family=binomial(link="probit"))` within tolerance 0.05.

2. **`probit_gibbs(X, y, n_iter, burn_in, beta0, Sigma0)`** — Returns a list with `beta_draws` (matrix), `posterior_mean` (numeric vector), and `posterior_sd` (numeric vector). For large N, posterior means should be close to MLE estimates.

3. **`probit_mh(X, y, n_iter, burn_in, scale, beta0, Sigma0, init)`** — Returns a list with `beta_draws` (matrix), `posterior_mean` (numeric vector), `posterior_sd` (numeric vector), and `acceptance_rate` (scalar between 0 and 1). The acceptance rate should be between 0.20 and 0.50 with default settings.

---

## 2. Test Infrastructure

### 2.1 Test Runner

File: `tests/testthat.R`
```r
library(testthat)
library(exampleProbit)
test_check("exampleProbit")
```

### 2.2 Test File Organization

| Test file | Scope |
|-----------|-------|
| `tests/testthat/test-probit_mle.R` | MLE correctness, edge cases |
| `tests/testthat/test-probit_gibbs.R` | Gibbs correctness, posterior properties |
| `tests/testthat/test-probit_mh.R` | MH correctness, acceptance rate |
| `tests/testthat/test-simulation.R` | Simulation result validation (acceptance criteria C1-C7) |

---

## 3. MLE Test Scenarios — test-probit_mle.R

### 3.1 Benchmark Against glm()

**Setup** (use in all MLE tests):
```r
set.seed(42)
N <- 1000
x1 <- rnorm(N)
X <- cbind(1, x1)
beta_true <- c(-1, 0.5)
prob <- pnorm(X %*% beta_true)
y <- rbinom(N, 1, prob)
```

**Scenario 3.1.1**: MLE coefficients match glm

- Run: `fit <- probit_mle(X, y)`
- Run: `glm_fit <- glm(y ~ x1, family = binomial(link = "probit"))`
- Assert: `abs(fit$coefficients[1] - coef(glm_fit)[1]) < 0.05`
- Assert: `abs(fit$coefficients[2] - coef(glm_fit)[2]) < 0.05`
- This verifies acceptance criterion **C1**.

**Scenario 3.1.2**: MLE standard errors match glm

- Assert: `abs(fit$se[1] - summary(glm_fit)$coefficients[1,2]) < 0.01`
- Assert: `abs(fit$se[2] - summary(glm_fit)$coefficients[2,2]) < 0.01`

**Scenario 3.1.3**: MLE log-likelihood matches glm

- Assert: `abs(fit$loglik - as.numeric(logLik(glm_fit))) < 0.01`

**Scenario 3.1.4**: Variance-covariance matrix properties

- Assert: `vcov` is a 2x2 matrix.
- Assert: `vcov` is symmetric: `max(abs(fit$vcov - t(fit$vcov))) < 1e-10`.
- Assert: All diagonal elements are positive.
- Assert: `fit$se` equals `sqrt(diag(fit$vcov))` within 1e-10.

### 3.2 Return Structure

**Scenario 3.2.1**: Return type and names

- Assert: Result is a list.
- Assert: Names include `"coefficients"`, `"vcov"`, `"se"`, `"loglik"`.
- Assert: `coefficients` has length k (= ncol(X)).
- Assert: `vcov` has dimensions k x k.
- Assert: `se` has length k.
- Assert: `loglik` is a single numeric value (not NA, not NaN).

### 3.3 Multiple Sample Sizes

**Scenario 3.3.1**: MLE works at N = 200

- Use seed 42, N = 200, same DGP.
- Assert: Coefficients within 0.05 of glm.
- Assert: No errors or warnings.

**Scenario 3.3.2**: MLE works at N = 5000

- Use seed 42, N = 5000, same DGP.
- Assert: Coefficients within 0.01 of glm (tighter at large N).

### 3.4 Convergence Warning

**Scenario 3.4.1**: Warning on non-convergence

- Run with max_iter = 1.
- Expect a warning message containing "converge" (case-insensitive).

---

## 4. Gibbs Sampler Test Scenarios — test-probit_gibbs.R

### 4.1 Posterior Properties

**Setup**:
```r
set.seed(42)
N <- 1000
x1 <- rnorm(N)
X <- cbind(1, x1)
beta_true <- c(-1, 0.5)
y <- rbinom(N, 1, pnorm(X %*% beta_true))
```

**Scenario 4.1.1**: Posterior means close to MLE (large N)

- Run: `gibbs <- probit_gibbs(X, y, n_iter = 3500, burn_in = 500)`
- Run: `mle <- probit_mle(X, y)`
- Assert: `abs(gibbs$posterior_mean[1] - mle$coefficients[1]) < 0.1`
- Assert: `abs(gibbs$posterior_mean[2] - mle$coefficients[2]) < 0.1`
- This verifies acceptance criterion **C2** (Gibbs part).

**Scenario 4.1.2**: Posterior standard deviations are positive and reasonable

- Assert: All `gibbs$posterior_sd` > 0.
- Assert: All `gibbs$posterior_sd` < 1.0 (they should be small for N=1000).

**Scenario 4.1.3**: Beta draws matrix has correct dimensions

- Assert: `nrow(gibbs$beta_draws) == 3500 - 500` (= 3000 post-burn-in draws).
- Assert: `ncol(gibbs$beta_draws) == 2` (= k).

### 4.2 Prior Sensitivity

**Scenario 4.2.1**: Tight prior pulls posterior toward prior mean

- Run with tight prior: `gibbs_tight <- probit_gibbs(X, y, beta0 = c(0, 0), Sigma0 = diag(0.01, 2))`
- Assert: `abs(gibbs_tight$posterior_mean[1]) < abs(mle$coefficients[1])` — posterior mean for intercept is pulled toward 0.
- (With Sigma_0 = 0.01*I, the prior dominates, so posterior should be near 0.)

**Scenario 4.2.2**: Diffuse prior gives results close to MLE

- Run with very diffuse prior: `gibbs_diffuse <- probit_gibbs(X, y, beta0 = c(0, 0), Sigma0 = diag(10000, 2))`
- Assert: `abs(gibbs_diffuse$posterior_mean[1] - mle$coefficients[1]) < 0.15`

### 4.3 Return Structure

**Scenario 4.3.1**: Return type and names

- Assert: Result is a list.
- Assert: Names include `"beta_draws"`, `"posterior_mean"`, `"posterior_sd"`.
- Assert: Types are as documented.

### 4.4 Default Prior

**Scenario 4.4.1**: NULL prior arguments use defaults

- `gibbs_default <- probit_gibbs(X, y)` should run without error.
- `gibbs_explicit <- probit_gibbs(X, y, beta0 = rep(0, 2), Sigma0 = diag(100, 2))` should give similar results.
- Assert: `max(abs(gibbs_default$posterior_mean - gibbs_explicit$posterior_mean)) < 0.1` (same seed, same result).

Note: For this test, both calls must use the same RNG state. Use `set.seed(123)` before each call.

---

## 5. Metropolis-Hastings Test Scenarios — test-probit_mh.R

### 5.1 Acceptance Rate

**Setup**:
```r
set.seed(42)
N <- 1000
x1 <- rnorm(N)
X <- cbind(1, x1)
beta_true <- c(-1, 0.5)
y <- rbinom(N, 1, pnorm(X %*% beta_true))
```

**Scenario 5.1.1**: Acceptance rate in target range

- Run: `mh <- probit_mh(X, y, n_iter = 10000, burn_in = 2000, scale = 1.0)`
- Assert: `mh$acceptance_rate >= 0.20`
- Assert: `mh$acceptance_rate <= 0.50`
- This verifies acceptance criterion **C3**.

### 5.2 Posterior Properties

**Scenario 5.2.1**: Posterior means close to MLE (large N)

- Run: `mle <- probit_mle(X, y)`
- Assert: `abs(mh$posterior_mean[1] - mle$coefficients[1]) < 0.1`
- Assert: `abs(mh$posterior_mean[2] - mle$coefficients[2]) < 0.1`
- This verifies acceptance criterion **C2** (MH part).

**Scenario 5.2.2**: Beta draws matrix has correct dimensions

- Assert: `nrow(mh$beta_draws) == 10000 - 2000` (= 8000 post-burn-in draws).
- Assert: `ncol(mh$beta_draws) == 2`.

### 5.3 Scale Sensitivity

**Scenario 5.3.1**: Very small scale gives high acceptance

- Run: `mh_small <- probit_mh(X, y, scale = 0.01)`
- Assert: `mh_small$acceptance_rate > 0.90` — nearly all proposals accepted (tiny steps).

**Scenario 5.3.2**: Very large scale gives low acceptance

- Run: `mh_large <- probit_mh(X, y, scale = 100.0)`
- Assert: `mh_large$acceptance_rate < 0.10` — most proposals rejected (huge jumps).

### 5.4 MLE Initialization

**Scenario 5.4.1**: Default init uses MLE

- Run with no init: `mh1 <- probit_mh(X, y)` — should not error.
- Run with explicit MLE init: `mle <- probit_mle(X, y); mh2 <- probit_mh(X, y, init = mle$coefficients)` — should not error.
- Both should produce valid results (posterior means finite, acceptance rate in (0,1)).

### 5.5 Return Structure

**Scenario 5.5.1**: Return type and names

- Assert: Result is a list.
- Assert: Names include `"beta_draws"`, `"posterior_mean"`, `"posterior_sd"`, `"acceptance_rate"`.

---

## 6. Edge Case Scenarios

### 6.1 Input Validation

**Scenario 6.1.1**: Mismatched dimensions

- `X` has 100 rows, `y` has 50 elements.
- Expect error (use `expect_error()`).

**Scenario 6.1.2**: Non-binary y

- `y` contains a value of 0.5.
- Expect error.

**Scenario 6.1.3**: Single-column X (intercept only)

- `X <- matrix(1, nrow = 200, ncol = 1)`, `y` binary.
- All three functions should run without error and return k=1 results.

### 6.2 Small Sample

**Scenario 6.2.1**: Minimum viable N

- N = 10, k = 2.
- All three functions should run without error (may have poor estimates but should not crash).

### 6.3 All-ones or All-zeros Response

**Scenario 6.3.1**: All y = 1

- MLE should either converge to large coefficients or issue a warning.
- Gibbs and MH should still run (prior regularizes).

---

## 7. Cross-Reference Benchmarks

### 7.1 glm as Reference

For any dataset, `probit_mle` coefficients must match `glm(y ~ . - 1, data, family = binomial(link = "probit"))` within 0.05 (C1).

The `-1` in the formula suppresses glm's automatic intercept since X already contains the intercept column.

### 7.2 Bayesian-Frequentist Convergence

For N >= 500 with diffuse prior (Sigma_0 = 100*I), Gibbs and MH posterior means must be within 0.1 of MLE (C2).

---

## 8. Timing Tests

**Scenario 8.1**: MLE faster than Gibbs

- Run MLE and Gibbs each on the same dataset (N=1000, seed 42).
- Time each using `system.time()`.
- Assert: MLE elapsed time < Gibbs elapsed time / 5 (C5: MLE at least 5x faster).
- Note: Use `\donttest{}` wrapper in examples if timing tests are sensitive to CI load.

---

## 9. Simulation Validation — test-simulation.R

This section validates the Monte Carlo simulation results against acceptance criteria C1-C7. Tester runs the simulation harness (produced by simulator) and checks outputs.

### 9.1 Prerequisites

- The simulation script must be at `inst/simulation/run_simulation.R`.
- It must be source-able and produce a results object or file.

### 9.2 Acceptance Criteria Assertions

After running the simulation with the design specified (4 sample sizes, 3 methods, 500 reps):

**C1 — MLE matches glm**:
- For each replication, the simulation already compares MLE to glm. Verify that for all N, the average absolute difference between MLE and glm coefficients is < 0.05 across replications.

**C2 — Bayesian-Frequentist agreement**:
- For N >= 500: verify mean absolute difference between Gibbs posterior mean and MLE < 0.1.
- For N >= 500: verify mean absolute difference between MH posterior mean and MLE < 0.1.

**C3 — MH acceptance rate**:
- For all N: verify mean acceptance rate across replications is in [0.20, 0.50].

**C4 — Coverage**:
- For all methods, all N, both beta coefficients: verify that empirical 95% CI coverage is between 0.90 and 0.99.
- MLE CI: beta_hat_j +/- 1.96 * SE_j.
- Gibbs/MH CI: 2.5% and 97.5% quantiles of posterior draws.

**C5 — Speed**:
- For all N: verify MLE mean time < Gibbs mean time / 5.

**C6 — R CMD check**:
- Tester runs `R CMD check --as-cran` on the built package.
- Assert: 0 errors, 0 warnings.

**C7 — Package installability**:
- Tester runs `R CMD INSTALL` and verifies all three functions are callable from R.
- Assert: `library(exampleProbit)` loads without error.
- Assert: `probit_mle`, `probit_gibbs`, `probit_mh` are accessible (exist as functions).

### 9.3 Simulation Result Format

The simulation harness should produce a data frame or list with columns/fields:
- method: {"MLE", "Gibbs", "MH"}
- N: sample size
- beta_j: which coefficient (0 = intercept, 1 = slope)
- bias: mean bias across replications
- rmse: RMSE across replications
- coverage: empirical coverage proportion
- mean_time: average computation time (seconds)
- acceptance_rate: (MH only) mean acceptance rate

Tester reads this output and applies the assertions above.

---

## 10. Validation Commands

| Stage | Command |
|-------|---------|
| Unit tests | `Rscript -e "devtools::test()"` |
| Build | `R CMD build .` |
| Full check | `R CMD check --as-cran <tarball>` |
| Install | `R CMD INSTALL <tarball>` |
| Load test | `Rscript -e "library(exampleProbit); probit_mle; probit_gibbs; probit_mh"` |

Tester MUST run all validation commands and report results in `audit.md`.

---

## 11. Property-Based Invariants

### 11.1 MLE

- The log-likelihood at the MLE must be >= the log-likelihood at beta = 0 (the MLE improves on the null).
- SE values must be positive.
- vcov must be positive definite (all eigenvalues > 0).

### 11.2 Gibbs

- All posterior draws must be finite (no NaN, no Inf).
- posterior_mean must equal colMeans(beta_draws) within 1e-10.
- posterior_sd must equal apply(beta_draws, 2, sd) within 1e-10.

### 11.3 MH

- acceptance_rate must be in [0, 1].
- All posterior draws must be finite.
- If acceptance_rate is 0, all rows of beta_draws are identical (all proposals rejected).

---

## 12. CRAN Compliance Checks

Tester MUST verify (per r-package profile):

1. No use of `T`/`F` as logical values in R code.
2. No hardcoded `set.seed()` in package functions (only in tests/examples).
3. No bare `print()` or `cat()` in package functions (use `message()`).
4. No `options(warn = -1)`.
5. No `installed.packages()`.
6. No `<<-` or `.GlobalEnv` modification.
7. Examples complete in < 5 seconds each.
8. Package tarball < 5 MB.
