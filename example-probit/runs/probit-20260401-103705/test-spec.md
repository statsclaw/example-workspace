# Test Specification (test-spec.md)

```yaml
Request ID: probit-20260401-103705
Pipeline: Test (tester only)
Profile: r-package
Package: exampleProbit
```

---

## 1. Behavioral Contract

The `exampleProbit` package provides three probit model estimation functions:

1. **`probit_mle(X, y, ...)`** — returns MLE estimates via Newton-Raphson with coefficients, variance-covariance matrix, standard errors, log-likelihood, iteration count, and convergence flag
2. **`probit_gibbs(X, y, ...)`** — returns Bayesian posterior draws via Gibbs sampling with posterior mean and standard deviation
3. **`probit_mh(X, y, ...)`** — returns Bayesian posterior draws via Metropolis-Hastings with posterior mean, standard deviation, and acceptance rate

All three must produce consistent estimates of the probit model parameters for well-specified data. They must handle standard inputs without error and return correctly structured output.

---

## 2. Unit Test Scenarios

### 2.1 MLE Tests (`tests/testthat/test-probit_mle.R`)

**T1: Output structure**

- Input: X = cbind(1, rnorm(200)), y = rbinom(200, 1, 0.5), seed = 123
- Expected: Result is a list with names exactly: `coefficients`, `vcov`, `se`, `loglik`, `iterations`, `converged`
- Check types: coefficients is numeric vector of length 2, vcov is 2x2 matrix, se is numeric vector of length 2, loglik is scalar double, iterations is scalar integer, converged is scalar logical

**T2: Agreement with glm (Acceptance Criterion C1)**

- Input: Generate data with seed = 42, N = 500, X = cbind(1, rnorm(N)), true beta = (-1, 0.5), y = rbinom(N, 1, pnorm(X %*% beta_true))
- Reference: `glm(y ~ X[,2], family = binomial(link = "probit"))`
- Expected: max(abs(probit_mle(X, y)$coefficients - coef(glm_fit))) < 0.05
- Tolerance: 0.05 absolute difference per coefficient (C1)

**T3: Convergence**

- Input: Same as T2
- Expected: result$converged == TRUE
- Expected: result$iterations <= 100 (within max_iter)

**T4: Log-likelihood is finite and negative**

- Input: Same as T2
- Expected: is.finite(result$loglik) and result$loglik < 0

**T5: Standard errors are positive**

- Input: Same as T2
- Expected: all(result$se > 0)

**T6: Variance-covariance matrix is symmetric positive definite**

- Input: Same as T2
- Expected: isSymmetric(result$vcov) is TRUE
- Expected: all eigenvalues of result$vcov are > 0

**T7: Large sample accuracy**

- Input: seed = 42, N = 5000, X = cbind(1, rnorm(N)), true beta = (-1, 0.5), y = rbinom(N, 1, pnorm(X %*% beta_true))
- Expected: max(abs(result$coefficients - c(-1, 0.5))) < 0.15
- Rationale: MLE is consistent; at N=5000, estimates should be near truth

### 2.2 Gibbs Tests (`tests/testthat/test-probit_gibbs.R`)

**T8: Output structure**

- Input: X = cbind(1, rnorm(200)), y = rbinom(200, 1, 0.5), seed = 123, n_iter = 1000, burn_in = 200
- Expected: Result is a list with names: `beta_draws`, `posterior_mean`, `posterior_sd`
- Check types: beta_draws is matrix with (n_iter - burn_in) rows and ncol(X) columns, posterior_mean is numeric vector of length ncol(X), posterior_sd is numeric vector of length ncol(X)

**T9: Posterior mean near MLE (Acceptance Criterion C2)**

- Input: seed = 42, N = 500, X = cbind(1, rnorm(N)), true beta = (-1, 0.5), y = rbinom(N, 1, pnorm(X %*% beta_true))
- Run MLE first to get reference coefficients
- Run Gibbs with n_iter = 3500, burn_in = 500
- Expected: max(abs(gibbs$posterior_mean - mle$coefficients)) < 0.1 (C2: within 0.1 for N >= 500)

**T10: Posterior draws have correct dimensions**

- Input: n_iter = 2000, burn_in = 500
- Expected: nrow(beta_draws) == 1500, ncol(beta_draws) == ncol(X)

**T11: Posterior standard deviations are positive**

- Input: Same as T9
- Expected: all(gibbs$posterior_sd > 0)

**T12: Custom prior**

- Input: Same as T9 but with beta0 = rep(0, 2), Sigma0 = diag(100, 2)
- Expected: Function runs without error and returns valid structure
- Expected: Posterior mean is a finite numeric vector

### 2.3 MH Tests (`tests/testthat/test-probit_mh.R`)

**T13: Output structure**

- Input: X = cbind(1, rnorm(200)), y = rbinom(200, 1, 0.5), seed = 123, n_iter = 2000, burn_in = 500
- Expected: Result is a list with names: `beta_draws`, `posterior_mean`, `posterior_sd`, `acceptance_rate`
- Check types: beta_draws is matrix, posterior_mean and posterior_sd are numeric vectors, acceptance_rate is scalar double in [0, 1]

**T14: Acceptance rate in range (Acceptance Criterion C3)**

- Input: seed = 42, N = 500, X = cbind(1, rnorm(N)), true beta = (-1, 0.5), y = rbinom(N, 1, pnorm(X %*% beta_true)), scale = 1.0, n_iter = 10000, burn_in = 2000
- Expected: acceptance_rate >= 0.20 and acceptance_rate <= 0.50 (C3)

**T15: Posterior mean near MLE (Acceptance Criterion C2)**

- Input: Same as T14
- Expected: max(abs(mh$posterior_mean - mle$coefficients)) < 0.1 (C2)

**T16: Custom initialization**

- Input: Same as T14 but with init = c(0, 0)
- Expected: Function runs without error and returns valid output

**T17: Posterior draws have correct dimensions**

- Input: n_iter = 5000, burn_in = 1000
- Expected: nrow(beta_draws) == 4000, ncol(beta_draws) == ncol(X)

---

## 3. Edge Case Scenarios

**E1: Minimum sample size**

- Input: N = 10, X = cbind(1, rnorm(10)), y with at least one 0 and one 1, seed = 99
- Expected: All three functions run without error. Results may be imprecise but finite.

**E2: All-ones or all-zeros outcome**

- Input: N = 100, X = cbind(1, rnorm(100)), y = rep(1, 100)
- Expected for MLE: Function either returns coefficients (possibly extreme) or signals non-convergence (converged = FALSE). Must not crash or produce NaN.
- Expected for Gibbs/MH: Function returns finite posterior draws (the prior regularizes).

**E3: Single covariate (intercept only)**

- Input: N = 200, X = matrix(1, nrow = 200, ncol = 1), y = rbinom(200, 1, 0.3), seed = 77
- Expected: k = 1, all functions return length-1 coefficient vectors. MLE coefficient should be close to qnorm(0.3).

**E4: Burn-in equals zero**

- Input: Gibbs with burn_in = 0, n_iter = 1000
- Expected: nrow(beta_draws) == 1000 (all draws kept)

**E5: Large k relative to N**

- Input: N = 50, k = 10, X = cbind(1, matrix(rnorm(50*9), 50, 9)), y with mixed 0/1, seed = 55
- Expected: Functions run without error. MLE may not converge; Bayesian methods should still produce finite draws due to prior regularization.

---

## 4. Property-Based Invariants

**P1: MLE log-likelihood monotonicity**

- The log-likelihood should not decrease across Newton-Raphson iterations (within numerical tolerance). This is a property of Newton-Raphson on a concave function.
- Test approach: Run MLE with max_iter = 1, 2, 3, ..., 10 and verify loglik is non-decreasing (tolerance 1e-6 for numerical noise).

**P2: Bayesian posterior concentrates**

- As N increases (100, 500, 2000), Gibbs posterior_sd for each coefficient should decrease.
- Test: Run Gibbs at N = 100 and N = 2000 with same seed structure. Verify posterior_sd at N=2000 < posterior_sd at N=100 for each coefficient.

**P3: MH acceptance rate responds to scale**

- Larger scale should produce lower acceptance rate (proposals too far from mode).
- Test: Run MH with scale = 0.1 and scale = 5.0 on same data (N = 500, seed = 42). Verify acceptance_rate(scale=0.1) > acceptance_rate(scale=5.0).

**P4: Coefficient symmetry under relabeling**

- If all y are flipped (0 <-> 1), the intercept should approximately negate.
- Test: Generate data, run MLE. Flip y. Run MLE again. Verify coefficients are approximately negated (tolerance 0.05 for finite-sample).

---

## 5. Cross-Reference Benchmarks

**B1: R's glm as reference for MLE**

- `glm(y ~ X[,2], family = binomial(link = "probit"))` is the gold standard
- MLE coefficients must match within 0.05 (C1)
- Standard errors should match within 0.02 (secondary check, not a hard gate)

**B2: Bayesian consistency**

- For large N (>= 500), Gibbs and MH posterior means must be within 0.1 of MLE (C2)
- This tests that Bayesian methods with diffuse prior converge to the MLE

---

## 6. Simulation Validation (Acceptance Criteria C1-C5)

The tester validates simulation output produced by the simulator. The simulation runs R = 500 replications per scenario across N = {200, 500, 1000, 5000} for three methods.

### 6.1 Simulation Acceptance Criteria

**C1: MLE matches glm**

- For each replication at each N, MLE coefficients must match glm coefficients within 0.05
- Validation: Compute max absolute difference across all replications. The median max difference should be < 0.05. Allow up to 5% of replications to exceed 0.05 due to numerical edge cases.

**C2: Bayesian-MLE agreement for N >= 500**

- For N in {500, 1000, 5000}: Gibbs and MH posterior means must be within 0.1 of MLE
- Validation: For each scenario with N >= 500, compute |Bias_Gibbs - Bias_MLE| and |Bias_MH - Bias_MLE| for each coefficient. Both must be < 0.1.

**C3: MH acceptance rate**

- For each scenario: mean acceptance rate across R replications must be between 0.20 and 0.50
- Validation: Check that the average acceptance_rate column in the results table is within [0.20, 0.50] for every N.

**C4: 95% CI coverage**

- For all methods, all coefficients, all N: coverage must be between 0.90 and 0.99
- Validation: Read the Coverage column from the simulation results table. Every entry must satisfy 0.90 <= Coverage <= 0.99.

**C5: MLE speed advantage**

- MLE average time must be at least 5x faster than Gibbs average time
- Validation: For each N, compute Time_Gibbs / Time_MLE. The ratio must be >= 5.0 for all N.

### 6.2 How Tester Validates Simulation Output

The simulator produces a results file (e.g., `simulation_results.rds` or a summary table in `simulation.md`). Tester must:

1. Read the simulation results
2. For each acceptance criterion (C1-C5), extract the relevant metric column
3. Assert the criterion is met using `expect_true()` or `expect_lte()` / `expect_gte()`
4. Report any violations with the specific N, method, and coefficient that failed

---

## 7. R CMD check Requirements (C6, C7)

**C6: R CMD check**

Run: `R CMD build .` then `R CMD check --as-cran <tarball>`

Required: 0 ERRORs, 0 WARNINGs. NOTEs should be reviewed and reported.

CRAN compliance checks per r-package profile:
- No `T`/`F` used as logicals
- No hardcoded `set.seed()` in non-test code
- No bare `print()`/`cat()` in exported functions
- `Authors@R` format in DESCRIPTION
- Title in Title Case
- Description is 2+ sentences
- All exported functions documented
- Examples present and runnable (short ones unwrapped, MCMC ones in `\donttest{}`)

**C7: Package installability**

Run: `R CMD INSTALL <tarball>`

Required: Installation succeeds without errors. After installation:
- `library(exampleProbit)` loads without error
- `probit_mle`, `probit_gibbs`, `probit_mh` are all callable
- A quick smoke test with toy data returns valid output for each function

---

## 8. Validation Commands

| Command | Purpose |
| --- | --- |
| `R CMD build .` | Build source tarball |
| `R CMD check --as-cran exampleProbit_0.1.0.tar.gz` | Full CRAN-style check (C6) |
| `R CMD INSTALL exampleProbit_0.1.0.tar.gz` | Install package (C7) |
| `Rscript -e "devtools::test()"` | Run testthat suite |
| `Rscript -e "devtools::document()"` | Regenerate NAMESPACE and man/ |

### Execution Order

1. `devtools::document()` -- ensure NAMESPACE and man/ are up to date
2. `R CMD build .` -- build the tarball
3. `R CMD check --as-cran <tarball>` -- full check
4. `devtools::test()` -- granular test output
5. Smoke test after install

---

## 9. Test Data Generation

All tests should use deterministic seeds for reproducibility.

| Test | Seed | N | True beta |
| --- | --- | --- | --- |
| T2, T3, T4, T5, T6 | 42 | 500 | (-1, 0.5) |
| T7 | 42 | 5000 | (-1, 0.5) |
| T9 | 42 | 500 | (-1, 0.5) |
| T14, T15 | 42 | 500 | (-1, 0.5) |
| E1 | 99 | 10 | (-1, 0.5) |
| E3 | 77 | 200 | NA (intercept only) |
| E5 | 55 | 50 | random k=10 |

Data generation procedure (standard across tests):

```r
set.seed(SEED)
X <- cbind(1, rnorm(N))
beta_true <- c(-1, 0.5)
y <- rbinom(N, 1, pnorm(X %*% beta_true))
```

---

## 10. Tolerance Summary

| Criterion | Metric | Tolerance |
| --- | --- | --- |
| C1 | MLE vs glm coefficient difference | < 0.05 absolute |
| C2 | Bayesian posterior mean vs MLE (N >= 500) | < 0.1 absolute |
| C3 | MH acceptance rate | [0.20, 0.50] |
| C4 | 95% CI coverage | [0.90, 0.99] |
| C5 | Time ratio (Gibbs / MLE) | >= 5.0 |
| C6 | R CMD check | 0 errors, 0 warnings |
| C7 | R CMD INSTALL | success |
| T7 | Large-sample MLE accuracy | < 0.15 per coefficient |
| P1 | Log-likelihood monotonicity | 1e-6 tolerance |

**CRITICAL: Tester must NEVER relax these tolerances to make a failing test pass. If a test fails, the correct response is BLOCK, not tolerance adjustment.**
