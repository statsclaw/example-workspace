# Audit Report: exampleProbit

```yaml
RequestID: REQ-20260331-223247
Pipeline: Test
Profile: r-package
Verdict: BLOCK
Signal: BLOCK
Route-to: builder
Reason: MH acceptance rate with default scale=1.0 is ~0.70, far above C3 upper bound of 0.50
```

---

## 1. Environment Info

| Component | Version |
|-----------|---------|
| R | 4.5.1 (2025-06-13) |
| Platform | aarch64-apple-darwin20 |
| OS | macOS Tahoe 26.5 |
| Compiler | Apple clang 17.0.0 |
| Rcpp | 1.1.1 |
| RcppArmadillo | 15.2.3.1 |
| testthat | 3.2.3 |

---

## 2. Validation Commands Run

| # | Command | Result |
|---|---------|--------|
| 1 | `R CMD INSTALL .` | SUCCESS - package installed with 0 errors, 0 warnings |
| 2 | `Rscript -e "library(exampleProbit); probit_mle; probit_gibbs; probit_mh"` | SUCCESS - all three functions callable |
| 3 | `Rscript -e "devtools::test('.')"` | 85 PASS, 1 FAIL (pre-test-fix); then 168 PASS, 5 FAIL (post-test-fix) |
| 4 | `R CMD build .` | SUCCESS - exampleProbit_0.1.0.tar.gz (0.03 MB) |
| 5 | `R CMD check --as-cran exampleProbit_0.1.0.tar.gz` | 2 ERRORs (test failures + LaTeX system issue), 1 WARNING (LaTeX), 4 NOTEs |
| 6 | Monte Carlo simulation (500 reps x 4 N x 3 methods) | SUCCESS - completed in ~23 minutes |

---

## 3. Test Bug Fixes Applied

Before running the full validation, tester identified and fixed three bugs in `tests/testthat/test-simulation.R` (test file only, no source code modified):

1. **Variable name mismatch**: Test expected `results` variable but simulation harness produces `sim_results`. Fixed: changed test to look for `sim_results`.
2. **Column name mismatch**: Test used `beta_j` column name but simulation harness uses `beta_idx`. Fixed: changed all `beta_j` references to `beta_idx`.
3. **Helper function ordering**: `gen_probit_data_sim()` was defined after the test_that block that uses it. Fixed: moved definition before usage.

These fixes corrected 7 spurious test failures (6 skips + 1 error) in the initial run. After fixes, only 5 genuine failures remain (all C3-related).

---

## 4. Per-Test Result Table

### 4.1 MLE Tests (test-probit_mle.R) -- 32/32 PASS

| Test | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|------|--------|----------|--------|-----------|------------|---------|
| 3.1.1: MLE coefs match glm (N=1000) | beta_0 diff | < 0.05 | < 0.05 | atol=0.05 | -- | PASS |
| 3.1.1: MLE coefs match glm (N=1000) | beta_1 diff | < 0.05 | < 0.05 | atol=0.05 | -- | PASS |
| 3.1.2: MLE SE match glm (N=1000) | SE_0 diff | < 0.01 | < 0.01 | atol=0.01 | -- | PASS |
| 3.1.2: MLE SE match glm (N=1000) | SE_1 diff | < 0.01 | < 0.01 | atol=0.01 | -- | PASS |
| 3.1.3: MLE loglik match glm (N=1000) | loglik diff | < 0.01 | < 0.01 | atol=0.01 | -- | PASS |
| 3.1.4: vcov properties | symmetry | < 1e-10 | < 1e-10 | atol=1e-10 | -- | PASS |
| 3.1.4: vcov properties | diag positive | all > 0 | all > 0 | exact | -- | PASS |
| 3.1.4: vcov properties | se = sqrt(diag(vcov)) | < 1e-10 | < 1e-10 | atol=1e-10 | -- | PASS |
| 3.2.1: Return structure | is list | TRUE | TRUE | exact | -- | PASS |
| 3.2.1: Return structure | has all names | TRUE | TRUE | exact | -- | PASS |
| 3.2.1: Return structure | coefs length k | 2 | 2 | exact | -- | PASS |
| 3.2.1: Return structure | vcov dim k x k | 2x2 | 2x2 | exact | -- | PASS |
| 3.2.1: Return structure | se length k | 2 | 2 | exact | -- | PASS |
| 3.2.1: Return structure | loglik numeric | TRUE | TRUE | exact | -- | PASS |
| 3.3.1: MLE at N=200 | beta_0 diff | < 0.05 | < 0.05 | atol=0.05 | -- | PASS |
| 3.3.1: MLE at N=200 | beta_1 diff | < 0.05 | < 0.05 | atol=0.05 | -- | PASS |
| 3.3.2: MLE at N=5000 | beta_0 diff | < 0.01 | < 0.01 | atol=0.01 | -- | PASS |
| 3.3.2: MLE at N=5000 | beta_1 diff | < 0.01 | < 0.01 | atol=0.01 | -- | PASS |
| 3.4.1: Non-convergence warning | warning msg | "converge" | matched | regex | -- | PASS |
| 11.1: loglik >= null loglik | loglik | >= null | >= null | exact | -- | PASS |
| 11.1: SE positive | all SE > 0 | TRUE | TRUE | exact | -- | PASS |
| 11.1: vcov positive definite | eigenvalues | all > 0 | all > 0 | exact | -- | PASS |
| 6.1.1: Mismatched dimensions | error | expected | raised | exact | -- | PASS |
| 6.1.2: Non-binary y | error | expected | raised | exact | -- | PASS |
| 6.1.3: Single-column X | coef length | 1 | 1 | exact | -- | PASS |
| 6.1.3: Single-column X | vcov dim | 1x1 | 1x1 | exact | -- | PASS |
| 6.1.3: Single-column X | se length | 1 | 1 | exact | -- | PASS |
| 6.2.1: N=10 minimum | no error | TRUE | TRUE | exact | -- | PASS |
| 6.3.1: All y=1 | no crash | TRUE | TRUE | exact | -- | PASS |

### 4.2 Gibbs Tests (test-probit_gibbs.R) -- 22/22 PASS

| Test | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|------|--------|----------|--------|-----------|------------|---------|
| 4.1.1: Posterior means close to MLE | beta_0 diff | < 0.1 | < 0.1 | atol=0.1 | -- | PASS |
| 4.1.1: Posterior means close to MLE | beta_1 diff | < 0.1 | < 0.1 | atol=0.1 | -- | PASS |
| 4.1.2: Posterior SD positive/reasonable | all > 0 | TRUE | TRUE | exact | -- | PASS |
| 4.1.2: Posterior SD positive/reasonable | all < 1.0 | TRUE | TRUE | exact | -- | PASS |
| 4.1.3: Beta draws dimensions | nrow | 3000 | 3000 | exact | -- | PASS |
| 4.1.3: Beta draws dimensions | ncol | 2 | 2 | exact | -- | PASS |
| 4.2.1: Tight prior pulls to prior | abs(intercept) < abs(MLE) | TRUE | TRUE | exact | -- | PASS |
| 4.2.2: Diffuse prior close to MLE | beta_0 diff | < 0.15 | < 0.15 | atol=0.15 | -- | PASS |
| 4.3.1: Return structure | is list, has names | TRUE | TRUE | exact | -- | PASS |
| 4.4.1: Default prior = explicit default | max diff | < 0.1 | < 0.1 | atol=0.1 | -- | PASS |
| 11.2: All draws finite | all finite | TRUE | TRUE | exact | -- | PASS |
| 11.2: posterior_mean = colMeans | max diff | < 1e-10 | < 1e-10 | atol=1e-10 | -- | PASS |
| 11.2: posterior_sd = apply sd | max diff | < 1e-10 | < 1e-10 | atol=1e-10 | -- | PASS |
| 6.1.3: Single-column X Gibbs | ncol | 1 | 1 | exact | -- | PASS |
| 6.2.1: N=10 Gibbs | no error | TRUE | TRUE | exact | -- | PASS |
| 6.3.1: All y=1 Gibbs | no error | TRUE | TRUE | exact | -- | PASS |

### 4.3 MH Tests (test-probit_mh.R) -- 27/28 PASS, 1 FAIL

| Test | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|------|--------|----------|--------|-----------|------------|---------|
| **5.1.1: Acceptance rate** | **rate** | **[0.20, 0.50]** | **0.702** | **exact range** | **+40.4%** | **FAIL** |
| 5.2.1: Posterior means close to MLE | beta_0 diff | < 0.1 | < 0.1 | atol=0.1 | -- | PASS |
| 5.2.1: Posterior means close to MLE | beta_1 diff | < 0.1 | < 0.1 | atol=0.1 | -- | PASS |
| 5.2.2: Beta draws dimensions | nrow | 8000 | 8000 | exact | -- | PASS |
| 5.2.2: Beta draws dimensions | ncol | 2 | 2 | exact | -- | PASS |
| 5.3.1: Small scale high acceptance | rate > 0.90 | TRUE | TRUE | exact | -- | PASS |
| 5.3.2: Large scale low acceptance | rate < 0.10 | TRUE | TRUE | exact | -- | PASS |
| 5.4.1: Default init no error | no error | TRUE | TRUE | exact | -- | PASS |
| 5.4.1: Explicit MLE init | finite means | TRUE | TRUE | exact | -- | PASS |
| 5.5.1: Return structure | is list, has names | TRUE | TRUE | exact | -- | PASS |
| 8.1: MLE faster than Gibbs by 5x | MLE < Gibbs/5 | TRUE | TRUE | exact | -- | PASS |
| 11.3: acceptance_rate in [0, 1] | rate in [0,1] | TRUE | TRUE | exact | -- | PASS |
| 11.3: All draws finite | all finite | TRUE | TRUE | exact | -- | PASS |
| 6.1.1: Mismatched dimensions MH | error | expected | raised | exact | -- | PASS |
| 6.1.2: Non-binary y MH | error | expected | raised | exact | -- | PASS |
| 6.1.3: Single-column X MH | ncol | 1 | 1 | exact | -- | PASS |
| 6.2.1: N=10 MH | no error | TRUE | TRUE | exact | -- | PASS |
| 6.3.1: All y=1 MH | no error | TRUE | TRUE (with warning) | exact | -- | PASS |

### 4.4 Simulation Tests (test-simulation.R) -- 77/82 PASS, 5 FAIL (after test bug fixes)

| Test | Metric | Expected | Actual | Tolerance | Verdict |
|------|--------|----------|--------|-----------|---------|
| Harness exists | file exists | TRUE | TRUE | exact | PASS |
| C7 functions callable | is.function | TRUE | TRUE (all 3) | exact | PASS |
| Simulation runs | data.frame produced | TRUE | TRUE (24 rows) | exact | PASS |
| C1: MLE bias all N | abs(bias) < 0.05 | TRUE | max 0.029 | atol=0.05 | PASS |
| C2: Gibbs-MLE (N>=500) | abs(bias_diff) < 0.1 | TRUE | max 0.003 | atol=0.1 | PASS |
| C2: MH-MLE (N>=500) | abs(bias_diff) < 0.1 | TRUE | max 0.003 | atol=0.1 | PASS |
| **C3: MH accept N=200** | **<= 0.50** | **<= 0.50** | **0.701** | **exact** | **FAIL** |
| **C3: MH accept N=500** | **<= 0.50** | **<= 0.50** | **0.698** | **exact** | **FAIL** |
| **C3: MH accept N=1000** | **<= 0.50** | **<= 0.50** | **0.698** | **exact** | **FAIL** |
| **C3: MH accept N=5000** | **<= 0.50** | **<= 0.50** | **0.698** | **exact** | **FAIL** |
| C4: Coverage all | [0.90, 0.99] | all in range | [0.926, 0.960] | exact | PASS |
| C5: MLE speed | < Gibbs/5 | TRUE | 500x+ ratio | exact | PASS |
| Reproducibility | identical | TRUE | TRUE | exact | PASS |

---

## 5. R CMD check Output Summary

**Command**: `R CMD check --as-cran exampleProbit_0.1.0.tar.gz`

**Result**: 2 ERRORs, 1 WARNING, 4 NOTEs

### Errors
1. **Test failures** (package issue): 5 test failures, all related to MH acceptance rate exceeding 0.50 (C3 criterion). Specifically:
   - test-probit_mh.R:24 -- acceptance_rate = 0.702, not < 0.50
   - test-simulation.R:121 -- C3 fails for N in {200, 500, 1000, 5000} (rates: 0.701, 0.698, 0.698, 0.698)

2. **PDF manual LaTeX** (system issue, NOT package): `inconsolata.sty` not found. This is a missing LaTeX package on the build system, not an R package defect.

### Warning
1. **LaTeX PDF generation** (system issue): Same root cause as error #2.

### Notes
1. New submission / maintainer info -- expected for new package
2. Unable to verify current time -- system clock issue
3. HTML tidy version -- system tool issue
4. Non-standard check directory files -- LaTeX intermediate file

**Assessment of C6**: Excluding system environment issues (LaTeX), the package itself has 0 errors from packaging, compilation, examples, documentation, and all non-test checks. The only ERROR is test failures, which are caused by the C3 source code defect. Once C3 is fixed, the R CMD check is expected to produce 0 errors, 0 warnings (the LaTeX issue is a build system configuration problem, not a package problem).

---

## 6. Full Simulation Results

### 6.1 Bias and RMSE

| Method | N | Coef | Bias | RMSE |
|--------|------|-----------|-----------|----------|
| MLE | 200 | intercept | -0.029016 | 0.120966 |
| MLE | 200 | slope | 0.016156 | 0.120745 |
| Gibbs | 200 | intercept | -0.038183 | 0.125592 |
| Gibbs | 200 | slope | 0.024304 | 0.124872 |
| MH | 200 | intercept | -0.038082 | 0.126332 |
| MH | 200 | slope | 0.024156 | 0.124589 |
| MLE | 500 | intercept | -0.007032 | 0.075494 |
| MLE | 500 | slope | 0.002693 | 0.074664 |
| Gibbs | 500 | intercept | -0.010277 | 0.076602 |
| Gibbs | 500 | slope | 0.005951 | 0.075628 |
| MH | 500 | intercept | -0.010190 | 0.076501 |
| MH | 500 | slope | 0.005468 | 0.075697 |
| MLE | 1000 | intercept | -0.005609 | 0.051323 |
| MLE | 1000 | slope | 0.006104 | 0.053428 |
| Gibbs | 1000 | intercept | -0.007211 | 0.051740 |
| Gibbs | 1000 | slope | 0.007643 | 0.053921 |
| MH | 1000 | intercept | -0.007173 | 0.051788 |
| MH | 1000 | slope | 0.007415 | 0.053884 |
| MLE | 5000 | intercept | -0.000641 | 0.022274 |
| MLE | 5000 | slope | 0.000718 | 0.024530 |
| Gibbs | 5000 | intercept | -0.000950 | 0.022332 |
| Gibbs | 5000 | slope | 0.001016 | 0.024620 |
| MH | 5000 | intercept | -0.000992 | 0.022312 |
| MH | 5000 | slope | 0.001019 | 0.024534 |

### 6.2 Coverage (95% CI)

| Method | N | Coef | Coverage |
|--------|------|-----------|----------|
| MLE | 200 | intercept | 0.956 |
| MLE | 200 | slope | 0.960 |
| Gibbs | 200 | intercept | 0.948 |
| Gibbs | 200 | slope | 0.956 |
| MH | 200 | intercept | 0.944 |
| MH | 200 | slope | 0.960 |
| MLE | 500 | intercept | 0.958 |
| MLE | 500 | slope | 0.960 |
| Gibbs | 500 | intercept | 0.952 |
| Gibbs | 500 | slope | 0.960 |
| MH | 500 | intercept | 0.954 |
| MH | 500 | slope | 0.958 |
| MLE | 1000 | intercept | 0.948 |
| MLE | 1000 | slope | 0.954 |
| Gibbs | 1000 | intercept | 0.946 |
| Gibbs | 1000 | slope | 0.952 |
| MH | 1000 | intercept | 0.946 |
| MH | 1000 | slope | 0.952 |
| MLE | 5000 | intercept | 0.948 |
| MLE | 5000 | slope | 0.926 |
| Gibbs | 5000 | intercept | 0.944 |
| Gibbs | 5000 | slope | 0.932 |
| MH | 5000 | intercept | 0.946 |
| MH | 5000 | slope | 0.926 |

### 6.3 Computation Time (seconds)

| Method | N | Mean Time (s) |
|--------|------|---------------|
| MLE | 200 | 0.000042 |
| MLE | 500 | 0.000112 |
| MLE | 1000 | 0.000226 |
| MLE | 5000 | 0.001132 |
| Gibbs | 200 | 0.022044 |
| Gibbs | 500 | 0.056532 |
| Gibbs | 1000 | 0.113376 |
| Gibbs | 5000 | 0.569544 |
| MH | 200 | 0.060080 |
| MH | 500 | 0.148620 |
| MH | 1000 | 0.296600 |
| MH | 5000 | 1.480060 |

Speed ratios (Gibbs/MLE): 525x (N=200), 505x (N=500), 502x (N=1000), 503x (N=5000).

### 6.4 MH Acceptance Rate

| N | Mean Acceptance Rate |
|------|---------------------|
| 200 | 0.7006 |
| 500 | 0.6983 |
| 1000 | 0.6985 |
| 5000 | 0.6976 |

### 6.5 Failure Rates

All methods: 0 failures out of 500 replications for all sample sizes (0% failure rate).

---

## 7. Simulation Validation Result Table

| Criterion | Metric | Target | Actual | At N | Threshold | Verdict |
|-----------|--------|--------|--------|------|-----------|---------|
| C1 | MLE abs(bias) | < 0.05 | 0.029 (max) | all | 0.05 | PASS |
| C2 | Gibbs-MLE bias diff | < 0.1 | 0.003 (max) | >= 500 | 0.1 | PASS |
| C2 | MH-MLE bias diff | < 0.1 | 0.003 (max) | >= 500 | 0.1 | PASS |
| **C3** | **MH accept rate** | **[0.20, 0.50]** | **0.698-0.701** | **all** | **[0.20, 0.50]** | **FAIL** |
| C4 | 95% CI coverage | [0.90, 0.99] | [0.926, 0.960] | all | [0.90, 0.99] | PASS |
| C5 | MLE/Gibbs speed ratio | >= 5x | 502-525x | all | 5x | PASS |
| C6 | R CMD check | 0 errors, 0 warnings | test failures (C3) + system LaTeX | -- | 0E, 0W | CONDITIONAL |
| C7 | Package installable, functions callable | TRUE | TRUE | -- | exact | PASS |

### Convergence Diagnostics

Bias decreases with N (consistency confirmed):
- MLE intercept bias: -0.029 (N=200) -> -0.001 (N=5000)
- MLE slope bias: 0.016 (N=200) -> 0.001 (N=5000)
- Same pattern for Gibbs and MH

RMSE decreases approximately as 1/sqrt(N) (confirmed):
- MLE intercept RMSE: 0.121 (N=200) -> 0.022 (N=5000) -- ratio 5.5x, sqrt(5000/200) = 5.0x
- MLE slope RMSE: 0.121 (N=200) -> 0.025 (N=5000) -- ratio 4.8x

Coverage approaches nominal level as N increases (confirmed for MLE; slight undercoverage at N=5000 for slope coefficient at 0.926 but still within [0.90, 0.99]).

### Reproducibility Check

Same seed (99) produces identical MLE coefficients and log-likelihood. PASS.

---

## 8. Before/After Comparison Table

N/A -- new feature, no prior implementation.

---

## 9. CRAN Compliance Checks

| Check | Result |
|-------|--------|
| 1. No T/F as logical values in R code | PASS |
| 2. No hardcoded set.seed() in package functions (only in examples) | PASS |
| 3. No bare print()/cat() in package functions | PASS |
| 4. No options(warn = -1) | PASS |
| 5. No installed.packages() | PASS |
| 6. No <<- or .GlobalEnv modification | PASS |
| 7. Examples complete in < 5 seconds | PASS (all examples ran OK) |
| 8. Package tarball < 5 MB | PASS (0.03 MB) |

---

## 10. C3 Failure Analysis

### Root Cause

The MH sampler uses proposal covariance `scale^2 * (X'X)^{-1}` with default `scale = 1.0`. For the probit model with the DGP used (beta_true = (-1, 0.5)), this produces a proposal distribution with steps that are too small, resulting in an acceptance rate of ~0.70 (well above the 0.50 upper bound).

### Evidence

Direct testing with different scale values:
- scale = 1.0: acceptance_rate = 0.702 (FAIL)
- scale = 2.0: acceptance_rate = 0.469 (PASS)
- scale = 3.0: acceptance_rate = 0.317 (PASS)

The acceptance rate is consistent across all sample sizes (0.698-0.701), confirming this is not a small-sample artifact but a systematic issue with the default scale parameter.

### Recommended Fix

Change the default value of `scale` in `probit_mh` from 1.0 to approximately 2.4 (targeting the theoretically optimal acceptance rate of ~0.234 for random-walk MH in high dimensions). Alternatively, scale = 2.0 gives 0.47 which is within the [0.20, 0.50] acceptance band.

The fix is in `src/probit_mh.cpp` line 30: change `double scale = 1.0` to `double scale = 2.4` (and update the R wrapper roxygen default accordingly).

---

## 11. Acceptance Criteria Summary

| Criterion | Description | Tolerance (from test-spec.md) | Tolerance Used | Verdict |
|-----------|-------------|-------------------------------|----------------|---------|
| C1 | MLE matches glm within 0.05 | atol = 0.05 | atol = 0.05 | **PASS** |
| C2 | Gibbs/MH within 0.1 of MLE (N >= 500) | atol = 0.1 | atol = 0.1 | **PASS** |
| C3 | MH acceptance rate in [0.20, 0.50] | range [0.20, 0.50] | range [0.20, 0.50] | **FAIL** |
| C4 | 95% CI coverage in [0.90, 0.99] | range [0.90, 0.99] | range [0.90, 0.99] | **PASS** |
| C5 | MLE at least 5x faster than Gibbs | ratio >= 5.0 | ratio >= 5.0 | **PASS** |
| C6 | R CMD check 0 errors, 0 warnings | 0E, 0W | 0E, 0W | **CONDITIONAL** |
| C7 | Package installs, functions callable | exact | exact | **PASS** |

Tolerances confirmation: All tolerances used match exactly what test-spec.md specifies. No tolerances were relaxed.

---

## 12. Overall Verdict

**BLOCK**

### Reason

Acceptance criterion C3 fails: MH acceptance rate with default scale=1.0 is approximately 0.70, which exceeds the specified upper bound of 0.50. This is a source code defect in `src/probit_mh.cpp` (default scale parameter too small).

C6 is conditional on C3: once the MH acceptance rate is fixed and the corresponding tests pass, R CMD check should produce 0 errors and 0 warnings from the package itself (the LaTeX errors are system environment issues, not package defects).

### Failure Routing

| Failure | Route To | Reason |
|---------|----------|--------|
| C3: MH acceptance rate > 0.50 | **builder** | Default `scale` parameter in `probit_mh()` is too small. Change default from 1.0 to ~2.4 in both `src/probit_mh.cpp` and `R/probit_mh.R` roxygen docs. The MH algorithm itself is correctly implemented -- it is only the default parameter value that needs adjustment. |

### What Passes

All other criteria (C1, C2, C4, C5, C7) pass convincingly. The estimators are numerically correct, Bayesian methods converge to frequentist results, coverage is nominal, and the package infrastructure is solid. Only the MH default scale needs adjustment.
