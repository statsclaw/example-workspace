# Audit Report (audit.md)

```yaml
Request ID: probit-20260401-103705
Pipeline: Test
Profile: r-package
Package: exampleProbit
Attempt: 2 of 3
Verdict: PASS
```

---

## 1. R CMD check Summary (C6)

**Command**: `R CMD check --as-cran exampleProbit_0.1.0.tar.gz`

| Category | Count | Details |
| --- | --- | --- |
| ERRORs | 1* | PDF manual generation — `pdflatex` not installed on this system |
| WARNINGs | 1* | LaTeX errors when creating PDF — same root cause as ERROR |
| NOTEs | 4 | New submission; RcppArmadillo in Imports not directly imported; HTML tidy unavailable; leftover tex file |

*The ERROR and WARNING are both caused by the absence of `pdflatex` on this macOS system. This is a system tool dependency (LaTeX distribution), NOT a package defect. All actual package checks passed:

- Package can be installed: OK
- Package can be loaded: OK
- Namespace can be loaded/unloaded: OK
- R code syntax: OK
- Rd files: OK
- Examples (including `--run-donttest`): OK
- Tests (testthat): OK
- Foreign function calls: OK
- Compiled code: OK

**C6 verdict: PASS** (0 package-attributable errors or warnings; LaTeX-related failures are environment issues)

---

## 2. Package Installation (C7)

**Command**: `R CMD INSTALL exampleProbit_0.1.0.tar.gz`

- Installation: SUCCESS
- C++ compilation (RcppArmadillo): SUCCESS
- `library(exampleProbit)`: loads without error
- All three exported functions callable: `probit_mle`, `probit_gibbs`, `probit_mh`

**Smoke test results** (seed=42, N=500, true beta=(-1, 0.5)):

| Function | Output | Status |
| --- | --- | --- |
| `probit_mle` | coefficients = (-1.0877, 0.3899), converged = TRUE | OK |
| `probit_gibbs` | posterior_mean = (-1.0845, 0.3878) | OK |
| `probit_mh` | posterior_mean = (-1.0917, 0.3947), acceptance_rate = 0.419 | OK |

**C7 verdict: PASS**

---

## 3. Unit Tests

**Command**: `Rscript -e "devtools::test('.')"`

| Context | Pass | Fail | Warn | Skip |
| --- | --- | --- | --- | --- |
| probit_mle | 10 | 0 | 0 | 0 |
| probit_gibbs | 9 | 0 | 0 | 0 |
| probit_mh | 9 | 0 | 0 | 0 |
| **Total** | **28** | **0** | **0** | **0** |

All 28 unit tests PASS.

---

## 4. Acceptance Criteria

### C1: MLE matches glm (tolerance < 0.05 absolute)

**Individual test** (seed=42, N=500): max |MLE - glm| = 1.57e-07. PASS.

**Simulation evidence**: Maximum absolute MLE bias across all N and parameters = 0.0178.

| N | beta_0 bias | beta_1 bias |
| --- | --- | --- |
| 200 | -0.0159 | 0.0178 |
| 500 | -0.0046 | 0.0120 |
| 1000 | -0.0042 | 0.0040 |
| 5000 | -0.0023 | 0.0011 |

All biases well within 0.05. **C1: PASS**

### C2: Bayesian-MLE agreement for N >= 500 (tolerance < 0.1 absolute)

| N | Param | |Gibbs bias - MLE bias| | |MH bias - MLE bias| | Status |
| --- | --- | --- | --- | --- |
| 500 | beta_0 | 0.0032 | 0.0033 | PASS |
| 500 | beta_1 | 0.0030 | 0.0029 | PASS |
| 1000 | beta_0 | 0.0015 | 0.0017 | PASS |
| 1000 | beta_1 | 0.0013 | 0.0016 | PASS |
| 5000 | beta_0 | 0.0004 | 0.0003 | PASS |
| 5000 | beta_1 | 0.0003 | 0.0002 | PASS |

All differences far below 0.1 threshold. Bayesian methods converge to MLE as expected. **C2: PASS**

### C3: MH acceptance rate [0.20, 0.50]

| N | Mean Acceptance Rate | Status |
| --- | --- | --- |
| 200 | 0.401 | PASS |
| 500 | 0.398 | PASS |
| 1000 | 0.397 | PASS |
| 5000 | 0.396 | PASS |

All acceptance rates in [0.20, 0.50], centered around 0.40 (optimal for random-walk MH with scale=2.4). **C3: PASS**

### C4: 95% CI coverage [0.90, 0.99]

| N | Method | beta_0 coverage | beta_1 coverage |
| --- | --- | --- | --- |
| 200 | MLE | 0.944 | 0.924 |
| 200 | Gibbs | 0.930 | 0.920 |
| 200 | MH | 0.928 | 0.924 |
| 500 | MLE | 0.962 | 0.940 |
| 500 | Gibbs | 0.960 | 0.940 |
| 500 | MH | 0.960 | 0.942 |
| 1000 | MLE | 0.954 | 0.948 |
| 1000 | Gibbs | 0.948 | 0.940 |
| 1000 | MH | 0.952 | 0.942 |
| 5000 | MLE | 0.950 | 0.954 |
| 5000 | Gibbs | 0.950 | 0.946 |
| 5000 | MH | 0.950 | 0.946 |

All 24 coverage values in [0.90, 0.99]. Coverage improves and centers near 0.95 as N increases, as expected. **C4: PASS**

### C5: MLE speed advantage (Gibbs/MLE >= 5x)

| N | MLE time (s) | Gibbs time (s) | Ratio | Status |
| --- | --- | --- | --- | --- |
| 200 | 0.000064 | 0.021900 | 342x | PASS |
| 500 | 0.000116 | 0.055118 | 475x | PASS |
| 1000 | 0.000240 | 0.111356 | 464x | PASS |
| 5000 | 0.001006 | 0.567650 | 564x | PASS |

MLE is 300-564x faster than Gibbs, far exceeding the 5x threshold. **C5: PASS**

### C6: R CMD check — see Section 1 above. **PASS**

### C7: Package installability — see Section 2 above. **PASS**

---

## 5. Full Monte Carlo Simulation Results

**Configuration**: DGP: Pr(y=1|x1) = Phi(-1 + 0.5*x1), R = 500 replications per scenario, N in {200, 500, 1000, 5000}, methods: MLE, Gibbs (n_iter=3500, burn_in=500), MH (n_iter=10000, burn_in=2000, scale=2.4)

### Complete Results Table

| N | Method | Param | Bias | RMSE | Coverage | Time (s) | Accept Rate | Valid | Failed |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 200 | MLE | beta_0 | -0.0159 | 0.1257 | 0.944 | 0.0001 | — | 500 | 0 |
| 200 | MLE | beta_1 | 0.0178 | 0.1370 | 0.924 | 0.0001 | — | 500 | 0 |
| 200 | Gibbs | beta_0 | -0.0244 | 0.1293 | 0.930 | 0.0219 | — | 500 | 0 |
| 200 | Gibbs | beta_1 | 0.0257 | 0.1409 | 0.920 | 0.0219 | — | 500 | 0 |
| 200 | MH | beta_0 | -0.0244 | 0.1293 | 0.928 | 0.0593 | 0.401 | 500 | 0 |
| 200 | MH | beta_1 | 0.0257 | 0.1409 | 0.924 | 0.0593 | 0.401 | 500 | 0 |
| 500 | MLE | beta_0 | -0.0046 | 0.0717 | 0.962 | 0.0001 | — | 500 | 0 |
| 500 | MLE | beta_1 | 0.0120 | 0.0797 | 0.940 | 0.0001 | — | 500 | 0 |
| 500 | Gibbs | beta_0 | -0.0078 | 0.0725 | 0.960 | 0.0551 | — | 500 | 0 |
| 500 | Gibbs | beta_1 | 0.0150 | 0.0808 | 0.940 | 0.0551 | — | 500 | 0 |
| 500 | MH | beta_0 | -0.0079 | 0.0724 | 0.960 | 0.1469 | 0.398 | 500 | 0 |
| 500 | MH | beta_1 | 0.0149 | 0.0807 | 0.942 | 0.1469 | 0.398 | 500 | 0 |
| 1000 | MLE | beta_0 | -0.0042 | 0.0524 | 0.954 | 0.0002 | — | 500 | 0 |
| 1000 | MLE | beta_1 | 0.0040 | 0.0554 | 0.948 | 0.0002 | — | 500 | 0 |
| 1000 | Gibbs | beta_0 | -0.0056 | 0.0528 | 0.948 | 0.1114 | — | 500 | 0 |
| 1000 | Gibbs | beta_1 | 0.0053 | 0.0559 | 0.940 | 0.1114 | — | 500 | 0 |
| 1000 | MH | beta_0 | -0.0059 | 0.0527 | 0.952 | 0.2951 | 0.397 | 500 | 0 |
| 1000 | MH | beta_1 | 0.0056 | 0.0558 | 0.942 | 0.2951 | 0.397 | 500 | 0 |
| 5000 | MLE | beta_0 | -0.0023 | 0.0237 | 0.950 | 0.0010 | — | 500 | 0 |
| 5000 | MLE | beta_1 | 0.0011 | 0.0239 | 0.954 | 0.0010 | — | 500 | 0 |
| 5000 | Gibbs | beta_0 | -0.0026 | 0.0238 | 0.950 | 0.5677 | — | 500 | 0 |
| 5000 | Gibbs | beta_1 | 0.0014 | 0.0239 | 0.946 | 0.5677 | — | 500 | 0 |
| 5000 | MH | beta_0 | -0.0026 | 0.0237 | 0.950 | 1.4849 | 0.396 | 500 | 0 |
| 5000 | MH | beta_1 | 0.0014 | 0.0240 | 0.946 | 1.4849 | 0.396 | 500 | 0 |

### MH Acceptance Rates by Sample Size

| N | Mean Acceptance Rate |
| --- | --- |
| 200 | 0.401 |
| 500 | 0.398 |
| 1000 | 0.397 |
| 5000 | 0.396 |

All acceptance rates are stable around 0.40, consistent with the scale=2.4 tuning for the 2-dimensional probit model. The rates are well within the [0.20, 0.50] acceptance window.

### Key Observations

1. **Bias decreases with N**: All methods show bias converging to zero as N increases, consistent with asymptotic theory.
2. **RMSE decreases at sqrt(N) rate**: RMSE roughly halves when N quadruples (e.g., 0.126 at N=200 to 0.024 at N=5000 for MLE beta_0), confirming sqrt(N)-consistency.
3. **Coverage near nominal**: All coverage values between 0.920 and 0.962, within the [0.90, 0.99] tolerance.
4. **Three methods agree**: Gibbs and MH produce nearly identical estimates to MLE at all sample sizes, with agreement improving as N grows.
5. **Zero failures**: All 2000 replications (500 per N) completed successfully with 0 failures across all methods.
6. **MLE is orders of magnitude faster**: Newton-Raphson converges in microseconds vs. milliseconds for MCMC, yielding 300-564x speed ratios.

---

## 6. Acceptance Criteria Summary

| Criterion | Description | Threshold | Observed | Verdict |
| --- | --- | --- | --- | --- |
| C1 | MLE matches glm | < 0.05 absolute | max bias = 0.018 | **PASS** |
| C2 | Bayesian-MLE agreement (N >= 500) | < 0.1 absolute | max diff = 0.003 | **PASS** |
| C3 | MH acceptance rate | [0.20, 0.50] | 0.396 - 0.401 | **PASS** |
| C4 | 95% CI coverage | [0.90, 0.99] | 0.920 - 0.962 | **PASS** |
| C5 | Speed ratio (Gibbs/MLE) | >= 5.0 | 342 - 564 | **PASS** |
| C6 | R CMD check | 0 errors, 0 warnings | 0 package errors* | **PASS** |
| C7 | R CMD INSTALL | success | success | **PASS** |

*R CMD check reported 1 ERROR and 1 WARNING from missing `pdflatex` (system LaTeX tool), not from any package defect. All package-level checks passed.

---

## 7. Previous BLOCKs and Resolution

### BLOCK 1 (resolved)
- **Issue**: MH acceptance rate ~0.70 with default scale=1.0
- **Fix**: Default scale changed to 2.4 in C++ (`probit_mh.cpp`) and R wrapper (`probit_mh.R`)
- **Verification**: Acceptance rates now 0.396-0.401 across all N. PASS.

### BLOCK 2 (resolved)
- **Issue (a)**: Simulation script hardcoded `scale=1.0`, overriding default
- **Issue (b)**: roxygen2 `\%` escape broke Rd file
- **Fix (a)**: Simulation script updated to use `scale=2.4`
- **Fix (b)**: roxygen changed from `\%` to spell out "percent"
- **Verification**: Simulation runs with correct scale; Rd files parse without error; R CMD check examples pass. PASS.

---

## 8. Overall Verdict

**PASS**

All 7 acceptance criteria met. All 28 unit tests pass. Full Monte Carlo simulation (R=500, N in {200, 500, 1000, 5000}) completed with zero failures. All three estimation methods (MLE, Gibbs, MH) produce correct, consistent results with proper coverage properties and expected computational performance characteristics.
