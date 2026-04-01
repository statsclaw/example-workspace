# Test Specification — Convergence Performance and Robustness Improvements

**Request ID**: issue-1-20260331-230107
**Profile**: r-package

---

## 1. Behavioral Contract

The convergence and robustness improvements MUST satisfy all of the following:

1. **No regression in estimate accuracy**: Point estimates (ATT, beta, factor, lambda) produced by the improved code MUST be numerically equivalent to those produced by the original code, within the convergence tolerance.
2. **Faster convergence**: Iteration counts (`niter`) on moderate-to-large panels MUST decrease or remain unchanged. They MUST NOT increase.
3. **Improved robustness**: Estimates MUST be more stable across different initial conditions and perturbations of tuning parameters.
4. **R CMD check clean**: Package MUST pass `R CMD check` with zero errors and zero warnings (NOTEs are acceptable).
5. **All existing tests pass**: Every test in `tests/testthat/` MUST continue to pass.
6. **API preservation**: No changes to user-facing function signatures or default parameter values.

---

## 2. Validation Commands

### Primary validation

```bash
cd /Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/example-fect
R CMD build . && R CMD check --no-manual --no-vignettes *.tar.gz
```

Expected: 0 errors, 0 warnings.

### Unit tests

```bash
cd /Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/example-fect
Rscript -e 'devtools::test()'
```

Expected: All existing tests pass.

### Compilation check

```bash
cd /Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/example-fect
Rscript -e 'devtools::load_all(); cat("Compilation OK\n")'
```

Expected: Clean compilation with no errors. Warnings about unused variables or deprecated features are acceptable if they existed before.

---

## 3. Test Scenarios

### Scenario 1: Estimate Accuracy Preservation (FE, no factors)

**Purpose**: Verify that the basic FE estimator (r=0) produces identical results after the changes.

```r
set.seed(42)
N <- 30; TT <- 15; T0 <- 10; Ntr <- 10; tau <- 3.0

alpha_i <- rnorm(N, 0, 1)
xi_t <- rnorm(TT, 0, 0.5)
simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = as.vector(sapply(1:N, function(i)
        sapply(1:TT, function(t)
            alpha_i[i] + xi_t[t] + tau * (i <= Ntr && t > T0) + rnorm(1)))),
    D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
)

out <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                  method = "fe", force = "two-way", se = FALSE, parallel = FALSE)
```

**Expected**:
- `out$att.avg` is within 1.0 of tau=3.0 (single replication, noisy)
- `out$niter` is a finite integer (no infinite loop)
- No errors or warnings

### Scenario 2: Estimate Accuracy Preservation (IFE, r=1)

**Purpose**: Verify IFE estimator with 1 factor.

```r
set.seed(101)
N <- 30; TT <- 15; T0 <- 10; Ntr <- 10; tau <- 2.0

alpha_i <- rnorm(N); xi_t <- rnorm(TT, 0, 0.5)
lambda_i <- rnorm(N); f_t <- rnorm(TT)
simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = as.vector(sapply(1:N, function(i)
        sapply(1:TT, function(t)
            alpha_i[i] + xi_t[t] + lambda_i[i]*f_t[t] +
            tau * (i <= Ntr && t > T0) + rnorm(1)))),
    D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
)

out <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                  method = "ife", r = 1, CV = FALSE,
                  force = "two-way", se = FALSE, parallel = FALSE)
```

**Expected**:
- `out$att.avg` is within 1.5 of tau=2.0
- `out$niter` is a finite positive integer
- No errors or warnings

### Scenario 3: Convergence Speed on Moderate Panel (KEY TEST)

**Purpose**: Verify iteration count decreases on moderate panel (N=200, T=30).

```r
set.seed(2001)
N <- 200; TT <- 30; T0 <- 20; Ntr <- 80; tau <- 1.5

alpha_i <- rnorm(N, 0, 2)
xi_t <- rnorm(TT, 0, 1)
lambda_i <- matrix(rnorm(N * 2), N, 2)
f_t <- matrix(rnorm(TT * 2), TT, 2)

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = as.vector(sapply(1:N, function(i)
        sapply(1:TT, function(t)
            alpha_i[i] + xi_t[t] + sum(lambda_i[i,] * f_t[t,]) +
            tau * (i <= Ntr && t > T0) + rnorm(1)))),
    D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
)

out <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                  method = "ife", r = 2, CV = FALSE,
                  force = "two-way", se = FALSE, parallel = FALSE)
```

**Expected**:
- `out$att.avg` is within 1.0 of tau=1.5
- `out$niter` <= 500 (must converge within max_iter=1000; ideally much less)
- Execution time should be reasonable (under 60 seconds)

**Acceptance criterion for convergence improvement**: Run this test and record `niter`. The improved code's `niter` MUST be <= the original code's `niter`. If the original converges in K iterations, the improved code should converge in at most K iterations (ideally fewer).

### Scenario 4: Convergence Speed on Large Panel

**Purpose**: Verify convergence on larger panel (N=500, T=50).

```r
set.seed(3001)
N <- 500; TT <- 50; T0 <- 30; Ntr <- 200; tau <- 1.0

alpha_i <- rnorm(N, 0, 2)
xi_t <- rnorm(TT, 0, 1)
lambda_i <- matrix(rnorm(N * 3), N, 3)
f_t <- matrix(rnorm(TT * 3), TT, 3)

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = as.vector(sapply(1:N, function(i)
        sapply(1:TT, function(t)
            alpha_i[i] + xi_t[t] + sum(lambda_i[i,] * f_t[t,]) +
            tau * (i <= Ntr && t > T0) + rnorm(1)))),
    D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
)

out <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                  method = "ife", r = 3, CV = FALSE,
                  force = "two-way", se = FALSE, parallel = FALSE,
                  tol = 1e-3, max.iteration = 1000)
```

**Expected**:
- `out$att.avg` is within 1.0 of tau=1.0
- `out$niter` <= 800 (must converge before max_iter; originally may hit max_iter)
- No errors

### Scenario 5: Robustness — Sensitivity to Initial Conditions

**Purpose**: Verify that different random seeds (which affect initialization through the data) produce more consistent estimates.

```r
set.seed(5001)
N <- 100; TT <- 20; T0 <- 12; Ntr <- 40; tau <- 2.0

estimates <- numeric(10)
for (k in 1:10) {
    set.seed(5000 + k)
    alpha_i <- rnorm(N, 0, 1.5)
    xi_t <- rnorm(TT, 0, 0.5)
    lambda_i <- matrix(rnorm(N * 2), N, 2)
    f_t <- matrix(rnorm(TT * 2), TT, 2)

    simdf <- data.frame(
        id = rep(1:N, each = TT),
        time = rep(1:TT, N),
        Y = as.vector(sapply(1:N, function(i)
            sapply(1:TT, function(t)
                alpha_i[i] + xi_t[t] + sum(lambda_i[i,] * f_t[t,]) +
                tau * (i <= Ntr && t > T0) + rnorm(1)))),
        D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
    )

    out <- suppressWarnings(fect::fect(
        Y ~ D, data = simdf, index = c("id", "time"),
        method = "ife", r = 2, CV = FALSE,
        force = "two-way", se = FALSE, parallel = FALSE
    ))
    estimates[k] <- out$att.avg
}
```

**Expected**:
- `mean(estimates)` is within 1.0 of tau=2.0
- `sd(estimates)` < 1.5 (estimates should not be wildly dispersed)
- No NA values in estimates

### Scenario 6: Robustness — Tolerance Sensitivity

**Purpose**: Verify that tightening tolerance does not dramatically change estimates.

```r
set.seed(6001)
N <- 60; TT <- 20; T0 <- 12; Ntr <- 20; tau <- 2.5

alpha_i <- rnorm(N, 0, 1)
xi_t <- rnorm(TT, 0, 0.5)
lambda_i <- rnorm(N)
f_t <- rnorm(TT)

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = as.vector(sapply(1:N, function(i)
        sapply(1:TT, function(t)
            alpha_i[i] + xi_t[t] + lambda_i[i]*f_t[t] +
            tau * (i <= Ntr && t > T0) + rnorm(1)))),
    D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
)

out_loose <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                        method = "ife", r = 1, CV = FALSE,
                        force = "two-way", se = FALSE, parallel = FALSE,
                        tol = 1e-3)

out_tight <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                        method = "ife", r = 1, CV = FALSE,
                        force = "two-way", se = FALSE, parallel = FALSE,
                        tol = 1e-5)
```

**Expected**:
- `abs(out_loose$att.avg - out_tight$att.avg)` < 0.3 (tightening tolerance should refine, not dramatically change)
- Both estimates within 1.5 of tau=2.5
- `out_tight$niter` >= `out_loose$niter` (tighter tolerance needs at least as many iterations)

### Scenario 7: Weighted Estimation with Burn-in

**Purpose**: Verify that the burn-in cap does not break weighted estimation.

```r
set.seed(7001)
N <- 50; TT <- 20; T0 <- 12; Ntr <- 20; tau <- 2.0

alpha_i <- rnorm(N); xi_t <- rnorm(TT, 0, 0.5)
lambda_i <- rnorm(N); f_t <- rnorm(TT)

Y_vec <- numeric(N * TT)
D_vec <- integer(N * TT)
W_vec <- numeric(N * TT)

idx <- 1
for (i in 1:N) {
    for (t in 1:TT) {
        treated <- (i <= Ntr) && (t > T0)
        D_vec[idx] <- as.integer(treated)
        Y_vec[idx] <- alpha_i[i] + xi_t[t] + lambda_i[i]*f_t[t] +
                      tau * D_vec[idx] + rnorm(1)
        W_vec[idx] <- runif(1, 0.5, 1.0)  # weights
        idx <- idx + 1
    }
}

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = Y_vec,
    D = D_vec,
    W = W_vec
)

out <- suppressWarnings(fect::fect(
    Y ~ D, data = simdf, index = c("id", "time"),
    method = "ife", r = 1, CV = FALSE,
    force = "two-way", se = FALSE, parallel = FALSE,
    weight = "W"
))
```

**Expected**:
- No errors or crashes
- `out$att.avg` is finite (not NA, not Inf)
- `out$niter` is a finite positive integer
- Burn-in completed successfully (no infinite loop from burn-in phase)

### Scenario 8: CFE Convergence

**Purpose**: Verify CFE estimation convergence with the improvements.

```r
set.seed(8001)
N <- 40; TT <- 15; T0 <- 10; Ntr <- 15; tau <- 2.0

alpha_i <- rnorm(N); xi_t <- rnorm(TT, 0, 0.5)
Z_i <- rnorm(N)  # time-invariant covariate

Y_vec <- numeric(N * TT)
D_vec <- integer(N * TT)
X1_vec <- numeric(N * TT)
Z_vec <- numeric(N * TT)

idx <- 1
for (i in 1:N) {
    for (t in 1:TT) {
        treated <- (i <= Ntr) && (t > T0)
        D_vec[idx] <- as.integer(treated)
        X1_vec[idx] <- rnorm(1)
        Z_vec[idx] <- Z_i[i]
        Y_vec[idx] <- alpha_i[i] + xi_t[t] + 0.5 * X1_vec[idx] +
                      tau * D_vec[idx] + rnorm(1)
        idx <- idx + 1
    }
}

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = Y_vec,
    D = D_vec,
    X1 = X1_vec,
    Z = Z_vec
)

out <- suppressWarnings(fect::fect(
    Y ~ D + X1, data = simdf, index = c("id", "time"),
    method = "cfe", r = 1, CV = FALSE,
    force = "two-way", se = FALSE, parallel = FALSE
))
```

**Expected**:
- No errors or crashes
- `out$att.avg` is within 1.5 of tau=2.0
- `out$niter` is a finite positive integer

### Scenario 9: Edge Case — r=0 (No Factors)

**Purpose**: Verify changes do not affect the r=0 code path.

```r
set.seed(9001)
N <- 30; TT <- 10; T0 <- 7; Ntr <- 10; tau <- 1.0

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = rnorm(N * TT) + rep(rnorm(N), each = TT) + rep(rnorm(TT), N) +
        tau * (rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0),
    D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
)

out <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                  method = "fe", force = "two-way", se = FALSE, parallel = FALSE)
```

**Expected**:
- No errors
- Result is identical to pre-change behavior (r=0 path should be unaffected by SVD warm-start and burn-in cap changes)

### Scenario 10: Edge Case — r=1, Small Panel

**Purpose**: Verify changes work correctly on minimal panel size.

```r
set.seed(10001)
N <- 10; TT <- 8; T0 <- 5; Ntr <- 4; tau <- 3.0

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = rnorm(N * TT) + rep(rnorm(N), each = TT) + rep(rnorm(TT), N) +
        rep(rnorm(N), each = TT) * rep(rnorm(TT), N) +
        tau * (rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0),
    D = as.integer(rep(1:N, each = TT) <= Ntr & rep(1:TT, N) > T0)
)

out <- fect::fect(Y ~ D, data = simdf, index = c("id", "time"),
                  method = "ife", r = 1, CV = FALSE,
                  force = "two-way", se = FALSE, parallel = FALSE)
```

**Expected**:
- No errors
- `out$att.avg` is finite
- `out$niter` is a finite positive integer

### Scenario 11: Balanced Panel (inter_fe path)

**Purpose**: Verify the balanced panel path (inter_fe / beta_iter) works correctly with the damped alternation.

```r
set.seed(11001)
N <- 50; TT <- 20; tau <- 2.0

alpha_i <- rnorm(N); xi_t <- rnorm(TT, 0, 0.5)
lambda_i <- rnorm(N); f_t <- rnorm(TT)
X1 <- matrix(rnorm(N * TT), TT, N)

Y <- matrix(0, TT, N)
for (i in 1:N) {
    for (t in 1:TT) {
        Y[t, i] <- alpha_i[i] + xi_t[t] + lambda_i[i]*f_t[t] + 0.5*X1[t,i] + rnorm(1)
    }
}

# Call the C++ function directly for balanced panel
X_cube <- array(X1, dim = c(TT, N, 1))
beta0 <- matrix(0, 1, 1)

out <- fect:::inter_fe(Y, X_cube, r = 1, force = 3L, beta0_in = beta0, tol = 1e-5, max_iter = 500L)
```

**Expected**:
- No errors
- `out$niter` is a finite positive integer less than 500
- `out$beta` is close to 0.5 (within 0.3)
- `out$sigma2` is positive and finite

---

## 4. Edge Case Scenarios

### Edge Case A: Empty Factor (validF=0)

When all interactive FE values are near zero, validF should be set to 0. Verify this still works.

```r
set.seed(20001)
N <- 20; TT <- 10

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = rnorm(N * TT) + rep(rnorm(N), each = TT) + rep(rnorm(TT), N),
    D = as.integer(rep(1:N, each = TT) <= 8 & rep(1:TT, N) > 7)
)

out <- suppressWarnings(fect::fect(
    Y ~ D, data = simdf, index = c("id", "time"),
    method = "ife", r = 1, CV = FALSE,
    force = "two-way", se = FALSE, parallel = FALSE
))
```

**Expected**: No errors. Result produced.

### Edge Case B: Convergence Near Zero Denominator

Verify the 1e-10 floor prevents division issues when fit_old is near zero.

```r
set.seed(20002)
N <- 15; TT <- 8

# Construct data where initial fit could be near zero
simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = rnorm(N * TT, 0, 0.01),  # very small values
    D = as.integer(rep(1:N, each = TT) <= 5 & rep(1:TT, N) > 5)
)

out <- suppressWarnings(fect::fect(
    Y ~ D, data = simdf, index = c("id", "time"),
    method = "ife", r = 1, CV = FALSE,
    force = "two-way", se = FALSE, parallel = FALSE
))
```

**Expected**: No NaN, no Inf, no division-by-zero errors. Result produced.

### Edge Case C: max.iteration = 1

Verify the algorithm handles a single iteration gracefully.

```r
set.seed(20003)
N <- 20; TT <- 10

simdf <- data.frame(
    id = rep(1:N, each = TT),
    time = rep(1:TT, N),
    Y = rnorm(N * TT) + rep(rnorm(N), each = TT),
    D = as.integer(rep(1:N, each = TT) <= 8 & rep(1:TT, N) > 7)
)

out <- suppressWarnings(fect::fect(
    Y ~ D, data = simdf, index = c("id", "time"),
    method = "ife", r = 1, CV = FALSE,
    force = "two-way", se = FALSE, parallel = FALSE,
    max.iteration = 1
))
```

**Expected**: No errors. Result produced (may not be accurate, but should not crash).

---

## 5. Property-Based Invariants

1. **Monotonicity of SSR**: Within any single run, the sum of squared residuals should be non-increasing across iterations (or at most very slightly increasing due to the E-step imputation). Tester can verify this by adding temporary logging to the C++ code if needed.

2. **Idempotency at convergence**: If the converged output is fed back as initial values, it should converge in 1 iteration (or very few).

3. **Factor orthogonality**: The factor matrix F from panel_factor should have orthogonal columns: F'F / T should be approximately diagonal. This property should be preserved by the warm-start SVD.

4. **Niter monotonicity with tolerance**: For the same data, tighter tolerance (smaller tol) should require at least as many iterations as looser tolerance.

---

## 6. Regression Scenarios

This is not a bug fix but an improvement. The regression test is: **all existing tests in `tests/testthat/` must continue to pass**. Key test files:

- `test-simulation-fe.R` — FE estimator under two-way FE DGP
- `test-simulation-ife.R` — IFE estimator under factor model DGP
- `test-simulation-cfe.R` — CFE estimator
- `test-simulation-staggered.R` — staggered adoption
- `test-fect-basic.R` — basic functionality
- `test-edge-cases.R` — edge cases
- `test-factors-from-refactor.R` — factor extraction
- `test-normalization.R` — normalization
- `test-book-claims.R` — published results

All must pass with zero failures.

---

## 7. Cross-Reference Benchmarks

The existing simulation tests (test-simulation-fe.R, test-simulation-ife.R, test-simulation-cfe.R) already implement Monte Carlo evaluations. They serve as the baseline benchmark. The improved code must:

- Achieve bias within the same tolerance bounds as the existing tests
- Achieve SD within the same bounds
- Produce no additional NA values in estimates

---

## 8. Acceptance Criteria Summary

| Criterion | Threshold | How to verify |
| --- | --- | --- |
| R CMD check | 0 errors, 0 warnings | `R CMD check` |
| Existing tests | All pass | `devtools::test()` |
| Compilation | Clean | `devtools::load_all()` |
| Accuracy (FE, r=0) | ATT within 1.0 of truth | Scenario 1 |
| Accuracy (IFE, r=1) | ATT within 1.5 of truth | Scenario 2 |
| Convergence (N=200) | niter <= original niter, niter < max_iter | Scenario 3 |
| Convergence (N=500) | niter <= original niter, niter < max_iter | Scenario 4 |
| Robustness (seeds) | SD of estimates < 1.5 | Scenario 5 |
| Robustness (tol) | Loose-tight ATT difference < 0.3 | Scenario 6 |
| Weighted estimation | No errors, finite result | Scenario 7 |
| CFE estimation | No errors, ATT within 1.5 | Scenario 8 |
| Edge: r=0 | No errors, identical behavior | Scenario 9 |
| Edge: small panel | No errors, finite result | Scenario 10 |
| Edge: near-zero values | No NaN/Inf | Edge Case B |
| Edge: max.iteration=1 | No crash | Edge Case C |
