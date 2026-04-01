# Comprehension Record

**Date**: 2026-03-31
**HOLD rounds used**: 0
**Verdict**: FULLY UNDERSTOOD

---

## Input Materials Read

1. `request.md` — scope and acceptance criteria for issue #1
2. `impact.md` — affected files, bottlenecks, robustness issues
3. `src/ife_sub.cpp` — core EM iteration loops (fe_ad_iter, fe_ad_covar_iter, fe_ad_inter_iter, fe_ad_inter_covar_iter, beta_iter)
4. `src/cfe_sub.cpp` — complex FE iteration (cfe_iter) with 5-component block coordinate descent
5. `src/ife.cpp` — main dispatch (inter_fe, inter_fe_ub) and initialization logic
6. `src/fe_sub.cpp` — SVD factor estimation (panel_factor), beta estimation (panel_beta), additive FE helpers (Y_demean, fe_add, ife)
7. `src/auxiliary.cpp` — E-step helpers (E_adj, wE_adj, FE_adj), panel_beta, XXinv
8. `src/fect.h` — C++ header with all function declarations
9. `R/fe.R` — R-level initialization (initialFit), parameter passing to C++
10. `R/cfe.R` — CFE R layer, input validation, calls to cfe_iter/complex_fe_ub
11. `R/default.R` — default parameters (tol=1e-3, max.iteration=1000)
12. Existing test files: test-simulation-fe.R, test-simulation-ife.R, test-simulation-cfe.R, test-fect-basic.R

---

## Core Requirement (Restated)

The example-fect package implements fixed effects counterfactual estimators for causal panel data analysis. The EM/optimization loop that iteratively estimates additive fixed effects, interactive fixed effects (via SVD), and regression coefficients converges slowly on moderate-to-large panels. Additionally, estimates are sensitive to initial conditions and tuning parameters. The goal is to improve convergence speed and robustness through targeted algorithmic improvements to the C++ core, without changing the user-facing R API or altering the statistical properties of the estimators.

---

## Algorithm Understanding

### The EM Framework

The package uses an EM-type algorithm for unbalanced panel data with missing entries. The observed outcome matrix is Y (T x N), with indicator matrix I (I_{it}=1 if observed). The model decomposes the fitted values as:

fit = mu + alpha_i + xi_t + X*beta + F*L' (for IFE models)

where:
- mu: scalar grand mean
- alpha_i: N x 1 unit fixed effects
- xi_t: T x 1 time fixed effects
- beta: p x 1 regression coefficients
- F: T x r factor matrix
- L: N x r loading matrix

### E-Step

For missing entries (I_{it}=0), the E-step imputes: YY_{it} = fit_{it} (current fitted value). For observed entries, YY_{it} = Y_{it}. With weights: YY_{it} = (1-W_{it})*fit_{it} + W_{it}*Y_{it}. Functions: `E_adj()` (unweighted) and `wE_adj()` (weighted).

### M-Step

The M-step estimates parameters from the completed matrix YY:

1. **Beta estimation** (`panel_beta`): OLS regression of (YY - FE) on X, using pre-computed (X'X)^{-1}.
2. **Additive FE estimation** (`Y_demean`, `fe_add`): Column means for unit FE, row means for time FE.
3. **Interactive FE estimation** (`panel_factor`): SVD of the demeaned residual matrix. If T < N, compute E*E'/(NT) then take top-r left singular vectors scaled by sqrt(T); otherwise compute E'*E/(NT) and take top-r right singular vectors scaled by sqrt(N).

### Convergence Criterion

The relative Frobenius norm: dif = ||fit_new - fit_old||_F / ||fit_old||_F. With weights: dif = ||W*(fit_new - fit_old)||_F / ||W*fit_old||_F. The loop terminates when dif < tol or niter > max_iter.

In fe_ad_inter_iter and fe_ad_inter_covar_iter, there is also a component-wise convergence check on the interactive FE component separately (max of overall and interactive-FE relative change).

### Burn-in Mechanism

When weights are used (use_weight=1), the algorithm first runs a "burn-in" phase with r_burnin = d - niter (where d = min(T,N)), which starts with many factors and gradually reduces. When this phase converges (dif <= tol) and niter <= d, the algorithm resets: stop_burnin=1, dif=1.0, niter=0, fit=Y0, and runs a second pass with the correct number of factors r. This ensures the weighted EM starts from a good initialization.

### beta_iter (Balanced Panel IFE)

For balanced panels with covariates and r>0: alternates between (1) computing FE = F*L', (2) estimating beta via panel_beta, (3) computing residuals U = Y - X*beta, (4) extracting factors via panel_factor(U, r). Convergence on ||beta_new - beta_old||_F < tol.

### cfe_iter (Complex FE)

Five-component block coordinate descent:
1. fit1: covariate fit (beta * X)
2. fit2: gamma components (time-varying coefficients of unit-level covariates)
3. fit3: kappa components (unit-varying coefficients of time-level covariates)
4. fit4: additive fixed effects (mu + alpha + xi)
5. fit5: interactive fixed effects (F * L')

Each iteration updates all five in sequence. Convergence requires ALL components to satisfy dif < tol, with an additional "overall_converged_count >= 3" stability check.

---

## Identified Bottlenecks and Root Causes

### Bottleneck 1: Burn-in Phase (HIGH impact)

In `fe_ad_inter_iter` (lines 287-294) and `fe_ad_inter_covar_iter` (lines 470-477), the burn-in forces TWO full convergence passes. The first pass uses r_burnin = d - niter factors (starting from d = min(T,N), which can be very large). When this first pass converges, everything resets to Y0 and a second pass runs. For large panels where d could be hundreds, the first pass does O(d) iterations each computing SVD with r_burnin factors — extremely expensive.

Similarly in `cfe_iter` (lines 473-481) the same burn-in pattern is used for the ife_part component.

**Improvement**: Cap r_burnin at a sensible maximum (e.g., min(d - niter, 5*r, 50)) instead of using d - niter which starts enormous. This preserves the initialization benefit while dramatically reducing per-iteration SVD cost.

### Bottleneck 2: SVD from Scratch Every Iteration (MEDIUM impact)

`panel_factor()` computes a full eigendecomposition of E*E'/(NT) or E'*E/(NT) every iteration. The SVD cost is O(T*N*r) for the matrix multiply plus O(min(T,N)^2 * max(T,N)) for the SVD. There is no warm-starting from the previous iteration's factors.

**Improvement**: Use the previous iteration's factors as an initial subspace and perform a few steps of subspace iteration (power method) before extracting the SVD. When the factors are already close to converged (which they are after the first few iterations), 1-2 power iterations suffice. This replaces a full SVD with a partial one. Create a `panel_factor_warm()` variant that accepts previous F and L as warm-start hints.

### Bottleneck 3: Naive Convergence Metric (MEDIUM impact)

The relative Frobenius norm dif = ||fit - fit_old||_F / ||fit_old||_F can be slow to satisfy because:
- When ||fit_old||_F is small (early iterations, near-zero effects), the denominator is tiny, making dif large even for small absolute changes. The code in fe_ad_inter_iter already adds 1e-10 to the denominator, but fe_ad_iter and fe_ad_covar_iter do NOT have this protection.
- The Frobenius norm averages over all T*N entries, so a few slowly-converging entries can drag the overall metric.

**Improvement**: (a) Add the 1e-10 floor to ALL convergence metric denominators for consistency. (b) Introduce an optional objective-function-based convergence check: monitor the sum of squared residuals (SSR). When the relative change in SSR is < tol, declare convergence. SSR-based convergence is standard for EM algorithms and tends to be more stable.

### Bottleneck 4: OLS Initialization (MEDIUM impact for robustness)

In `inter_fe()` (line 117), the starting value for beta is OLS/LSDV: beta0 = panel_beta(XX, invXX, YY, zeros). For `inter_fe_ub()`, the initial Y0 comes from `initialFit()` in R, which runs fastplm. When the true factor structure is complex (large r or strong factors), the OLS residuals may not capture the factor structure well, leading to poor initial factors and slow convergence.

**Improvement**: After the OLS initialization, perform a few "nuclear norm regularized" warm-up iterations with a larger number of factors (e.g., 2*r) to get a better initial subspace, then switch to the target r. This is a limited-cost improvement that helps the subsequent EM converge faster.

### Bottleneck 5: Sequential Multi-Component in CFE (LOW impact for this fix)

The CFE's five-component block coordinate descent is inherently sequential. However, the real cost is that the IFE component (fit5) runs a full SVD each iteration. The warm-start SVD improvement (Bottleneck 2) also applies here through `ife_part()`.

---

## Robustness Issues and Solutions

### Issue 1: Initial Condition Sensitivity

Root cause: OLS/LSDV initialization can produce fitted values far from the true factor structure, leading to different convergence basins depending on perturbations.

**Solution**: Better initialization via warm-up iterations (see Bottleneck 4 improvement). Also: add a multi-start option where the algorithm tries K=3 random initializations and picks the one with lowest final SSR. This is controlled by a new parameter `n.init` (default 1 for backward compatibility).

### Issue 2: Beta-Factor Oscillation

Root cause: In `beta_iter()`, beta and (F,L) are updated in strict alternation. When r is large relative to data, the two updates can "chase" each other. The convergence criterion is ||beta - beta_old||_F which only monitors beta, not the factors.

**Solution**: Add a damping/relaxation step to beta_iter: beta_new = (1-omega)*beta_old + omega*beta_proposed, with omega=0.8 as default. Also add a joint convergence check that monitors both beta change AND factor change. This prevents oscillation.

### Issue 3: Denominator Instability in Convergence Metric

Root cause: Missing 1e-10 floor in fe_ad_iter and fe_ad_covar_iter denominators.

**Solution**: Add consistent 1e-10 floor to all denominator calculations.

---

## Implicit Assumptions

1. The algorithms converge to a global optimum or at least a good local optimum — this is assumed by the EM framework but not guaranteed for the factor model setting.
2. The number of factors r is correctly specified (or selected by cross-validation elsewhere).
3. The observation pattern I has sufficient coverage for identification.
4. The weight matrix W has entries in [0,1] and is normalized (W = W/max(W) in inter_fe_ub).

---

## Intuition: Why These Improvements Work

The EM algorithm for factor models converges linearly at a rate proportional to the ratio of missing information to complete information. The improvements target the constant factor of this convergence rate:

- **Warm-start SVD**: The SVD at iteration k+1 is typically very close to iteration k. By starting from the previous subspace, we reduce the per-iteration SVD cost by 2-5x without changing the convergence path.
- **Burn-in capping**: The original burn-in with r_burnin = d - niter is designed to ensure the initial pass explores many possible factor structures. But using d (which can be hundreds) factors is overkill — 5*r or 50 factors is sufficient for initialization.
- **Damping**: Prevents the beta-factor alternation from overshooting, converting oscillatory behavior into monotone convergence.
- **SSR-based convergence**: Monitors the actual objective rather than a proxy (parameter change). This is the standard EM convergence criterion and is more robust to scale issues.

---

## Comprehension Self-Test

1. **Can I restate the core requirement?** Yes — see Core Requirement section above.
2. **Can I write out every formula?** Yes — all formulas verified against source code. The E-step, M-step (beta, additive FE, SVD factor extraction), convergence criterion, and burn-in logic are all documented above.
3. **Undefined symbols/concepts?** None. All symbols (Y, I, W, F, L, alpha, xi, mu, beta, VNT, r, force, tol) are defined.
4. **Judgment calls?** The choice of damping parameter omega=0.8 and burn-in cap of min(d-niter, 5*r, 50) are engineering decisions, not specified by the source. These are reasonable defaults used in the numerical optimization literature.
5. **Why does this work?** See Intuition section above.
6. **Implicit assumptions?** Listed above. All are standard for EM-based panel data methods.

---

## Final Verdict: FULLY UNDERSTOOD
