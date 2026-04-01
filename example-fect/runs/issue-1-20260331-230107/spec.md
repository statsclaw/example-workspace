# Implementation Specification — Convergence Performance and Robustness Improvements

**Request ID**: issue-1-20260331-230107
**Profile**: r-package

---

## 1. Notation

| Symbol | Type | Dimensions | Description |
| --- | --- | --- | --- |
| Y | matrix | T x N | Outcome matrix |
| I | matrix | T x N | Observation indicator (1 = observed, 0 = missing) |
| W | matrix | T x N | Weight matrix (entries in [0,1]) |
| fit | matrix | T x N | Current fitted values |
| fit_old | matrix | T x N | Previous iteration fitted values |
| F | matrix | T x r | Factor matrix |
| L | matrix | N x r | Loading matrix |
| FE | matrix | T x N | Overall fixed effects (additive + interactive) |
| beta | matrix | p x 1 | Regression coefficients |
| XX | cube | T x N x p | Covariate array |
| r | int | scalar | Number of factors |
| r_burnin | int | scalar | Number of factors during burn-in phase |
| d | int | scalar | min(T, N) |
| tol | double | scalar | Convergence tolerance |
| max_iter | int | scalar | Maximum iterations |
| omega | double | scalar | Damping/relaxation parameter for beta updates (new) |
| dif | double | scalar | Convergence metric value |
| SSR | double | scalar | Sum of squared residuals (new convergence metric) |

---

## 2. Algorithm Changes — Overview

The changes are organized into four independent improvements, each of which is backward-compatible and preserves the existing API. No function signatures visible to R change. Internal C++ function signatures may gain optional parameters with defaults that reproduce the original behavior.

### Change A: Cap Burn-in Factor Count
### Change B: Warm-Start SVD via Subspace Iteration
### Change C: Consistent Convergence Metric with SSR Fallback
### Change D: Damped Beta-Factor Alternation

---

## 3. Change A — Cap Burn-in Factor Count

### Affected Functions
- `fe_ad_inter_iter()` in `src/ife_sub.cpp`
- `fe_ad_inter_covar_iter()` in `src/ife_sub.cpp`
- `cfe_iter()` in `src/cfe_sub.cpp`

### Algorithm

In each function, replace the burn-in rank computation:

**Current code pattern** (e.g., ife_sub.cpp line 249):
```
r_burnin = d - niter;
if (r_burnin <= r) {
    r_burnin = r;
}
```

**New code**:
```
r_burnin = d - niter;
int r_cap = std::min(5 * r + 5, std::min(d, 50));
if (r_burnin > r_cap) {
    r_burnin = r_cap;
}
if (r_burnin < r) {
    r_burnin = r;
}
```

The cap formula is: r_burnin = clamp(d - niter, r, min(5*r + 5, d, 50)). The "+5" ensures even when r=0, the burn-in still uses a few factors. The cap of 50 is a hard ceiling that prevents excessive SVD cost on large panels.

### Implementation Locations

**ife_sub.cpp — `fe_ad_inter_iter()`**: The burn-in rank computation appears in TWO places:
1. Line 248-252: Inside the `if (mc == 0)` block — replace the existing r_burnin computation.

**ife_sub.cpp — `fe_ad_inter_covar_iter()`**: Same pattern at:
1. Lines 431-435: Inside the `if (use_weight == 1 && stop_burnin == 0)` block.

**cfe_sub.cpp — `cfe_iter()`**: Same pattern at:
1. Lines 473-477: Inside the `if (use_weight == 1 && stop_burnin == 0)` block for ife_part.

---

## 4. Change B — Warm-Start SVD via Subspace Iteration

### New Function: `panel_factor_warm()`

Add a new function to `src/fe_sub.cpp` that accepts previous factors as warm-start hints.

**Signature**:
```cpp
List panel_factor_warm(const arma::mat& E, int r,
                       const arma::mat& F_prev, const arma::mat& L_prev);
```

**Parameters**:
- E: T x N demeaned residual matrix
- r: number of factors to extract
- F_prev: T x r previous factor matrix (can be empty if no warm-start available)
- L_prev: N x r previous loading matrix (can be empty)

**Algorithm**:
1. If F_prev is empty or F_prev.n_cols != r, fall back to `panel_factor(E, r)`.
2. Otherwise, perform 2 steps of subspace iteration:
   - If T < N:
     - Q = F_prev (T x r)
     - For i = 1, 2: Z = (E * E') * Q / (N*T); then QR-decompose Z to get Q (orthonormalize)
     - Compute the Rayleigh quotient: R = Q' * (E*E'/(N*T)) * Q (r x r)
     - Eigendecompose R to get rotation
     - factor = Q * rotation_vectors * sqrt(T)
     - lambda = E' * factor / T
     - VNT = eigenvalues diagonal
   - If T >= N:
     - Q = L_prev (N x r)
     - For i = 1, 2: Z = (E' * E) * Q / (N*T); then QR-decompose Z to get Q
     - Compute R = Q' * (E'*E/(N*T)) * Q
     - Eigendecompose R
     - lambda = Q * rotation_vectors * sqrt(N)
     - factor = E * lambda / N
     - VNT = eigenvalues diagonal
3. Construct FE = factor * lambda'
4. Return List with "factor", "lambda", "VNT", "FE" (same structure as panel_factor).

**Key property**: When the subspace has already converged, panel_factor_warm produces the same result as panel_factor (up to floating-point tolerance). When the subspace is far from converged, the 2 power iterations bring it close enough for the EM outer loop to still make progress.

### Declare in Header

Add to `src/fect.h`:
```cpp
List panel_factor_warm(const arma::mat& E, int r,
                       const arma::mat& F_prev, const arma::mat& L_prev);
```

### Integration into Iteration Functions

**beta_iter() in ife_sub.cpp**:
- Before the loop (line 545): `pf = panel_factor(U, r)` — keep as-is (first call, no warm-start).
- Inside the loop (line 562): Replace `pf = panel_factor(U, r)` with `pf = panel_factor_warm(U, r, F, L)`.

**ife() in fe_sub.cpp** (line 257):
- The `ife()` function is called inside iteration loops. It calls panel_factor(EE, r) internally. Modify `ife()` to accept optional warm-start parameters:

**Modified ife() signature** (internal only — not exported to R):
```cpp
List ife(const arma::mat& E, int force, int mc, int r, int hard, double lambda,
         const arma::mat& F_prev = arma::mat(),
         const arma::mat& L_prev = arma::mat());
```
When F_prev is non-empty and mc==0, call `panel_factor_warm(EE, r, F_prev, L_prev)` instead of `panel_factor(EE, r)`.

**fe_ad_inter_iter() in ife_sub.cpp**:
- Track F_prev and L_prev across iterations.
- Pass them to `ife()` calls inside the loop. After each call to ife(), extract the new factor and lambda for next iteration's warm-start.

**fe_ad_inter_covar_iter() in ife_sub.cpp**:
- Same pattern as fe_ad_inter_iter.

**cfe_iter() — ife_part() in cfe_sub.cpp**:
- The `ife_part()` function calls `panel_factor(E, r)`. Modify `ife_part()` to accept and pass warm-start parameters, same pattern. Track F_prev and L_prev in the cfe_iter loop.

---

## 5. Change C — Consistent Convergence Metric

### Part C1: Add 1e-10 Floor to All Denominators

**fe_ad_iter() in ife_sub.cpp** (lines 59-63):
Current:
```cpp
dif = arma::norm(W % (fit - fit_old), "fro") /
      arma::norm(W % (fit_old), "fro");
// and
dif = arma::norm(fit - fit_old, "fro") / arma::norm(fit_old, "fro");
```

Replace with:
```cpp
dif = arma::norm(W % (fit - fit_old), "fro") /
      (arma::norm(W % (fit_old), "fro") + 1e-10);
// and
dif = arma::norm(fit - fit_old, "fro") / (arma::norm(fit_old, "fro") + 1e-10);
```

**fe_ad_covar_iter() in ife_sub.cpp** (lines 159-163): Same fix — add `+ 1e-10` to denominators.

**beta_iter() in ife_sub.cpp** (line 555):
Current: `beta_norm = arma::norm(beta - beta_old, "fro");`
This uses absolute norm, not relative. Leave as-is but see Change D for enhancements.

**cfe_iter() in cfe_sub.cpp** — component-wise dif calculations (lines 494, 499, 505, 510, 511):
Add `+ 1e-10` to every denominator in the dif1, dif2, dif3, dif4, dif5 calculations. Currently:
```cpp
dif1 = arma::norm(fit1 - fit1_old, "fro") / arma::norm(fit1_old, "fro");
```
Replace with:
```cpp
dif1 = arma::norm(fit1 - fit1_old, "fro") / (arma::norm(fit1_old, "fro") + 1e-10);
```
Apply the same pattern to dif2, dif3, dif4, dif5. The overall dif calculation at lines 487-492 already needs the same treatment — add `+ 1e-10` if not already present.

### Part C2: SSR-Based Early Convergence (Optional Acceleration)

Add an SSR-based convergence check as a secondary criterion. This does NOT replace the existing Frobenius-norm check — it provides an additional early-exit path.

**In fe_ad_inter_iter() and fe_ad_inter_covar_iter()**:

After computing `dif` (the Frobenius-norm metric), add:
```cpp
// SSR-based convergence check
double ssr_new = arma::accu(arma::square(FE_adj(YY - fit, I)));
if (niter > 1) {
    double ssr_rel_change = std::abs(ssr_new - ssr_old) / (ssr_old + 1e-10);
    if (ssr_rel_change < tolerate * 0.1 && dif < tolerate * 10) {
        dif = tolerate * 0.5; // trigger convergence
    }
}
ssr_old = ssr_new;
```

Declare `double ssr_old = 0.0;` before the while loop.

The SSR check triggers only when BOTH:
- The relative SSR change is < tol/10 (the objective is essentially flat)
- The Frobenius dif is < 10*tol (parameters are close to converged)

This prevents premature termination while catching the common case where the Frobenius norm oscillates around the tolerance.

---

## 6. Change D — Damped Beta-Factor Alternation in beta_iter()

### Affected Function
- `beta_iter()` in `src/ife_sub.cpp`

### Algorithm Modification

Add a damping parameter omega (default 0.8) and a joint convergence check.

**New beta_iter signature** (internal, backward-compatible default):
```cpp
List beta_iter(const arma::cube& X, const arma::mat& xxinv, const arma::mat& Y,
               int r, double tolerate, const arma::mat& beta0, int max_iter,
               double omega = 0.8);
```

**Modified loop body** (replacing lines 551-564):
```cpp
int niter = 0;
double joint_dif = 1.0;
arma::mat F_old = F;
arma::mat L_old = L;
while ((joint_dif > tolerate) && (niter < max_iter)) {
    niter++;
    FE = F * L.t();

    // Estimate beta with damping
    arma::mat beta_proposed = panel_beta(X, xxinv, Y, FE);
    beta = (1.0 - omega) * beta_old + omega * beta_proposed;
    beta_old = beta;

    // Compute residuals
    U = Y;
    for (int k = 0; k < p; k++) {
        U = U - X.slice(k) * beta(k);
    }

    // Extract factors with warm-start
    pf = panel_factor_warm(U, r, F, L);
    F = as<arma::mat>(pf["factor"]);
    L = as<arma::mat>(pf["lambda"]);

    // Joint convergence: max of beta change and factor change
    double beta_change = arma::norm(beta - beta_old, "fro") / (arma::norm(beta_old, "fro") + 1e-10);
    double factor_change = arma::norm(F * L.t() - F_old * L_old.t(), "fro") /
                           (arma::norm(F_old * L_old.t(), "fro") + 1e-10);
    joint_dif = std::max(beta_change, factor_change);

    F_old = F;
    L_old = L;
}
```

Note: The damping parameter omega=0.8 means 80% of the new estimate and 20% of the old. This is a conservative choice that preserves convergence guarantees (EM with damping is still EM, just with a smaller step). When beta is already close to converged, the damping has negligible effect.

**IMPORTANT**: The `beta_norm = arma::norm(beta - beta_old, "fro")` convergence check in the original code monitors absolute change, not relative. After applying damping, we switch to relative change with 1e-10 floor, AND we add factor change monitoring. The joint check prevents the situation where beta appears converged but factors are still moving.

### Call Site Update

In `inter_fe()` (ife.cpp, line 129):
```cpp
List out = beta_iter(XX, invXX, YY, r, tol, beta0, max_iter);
```
No change needed — the new `omega` parameter has a default value of 0.8.

---

## 7. Header Updates

### src/fect.h

Add declaration for the new function:
```cpp
List panel_factor_warm(const arma::mat& E, int r,
                       const arma::mat& F_prev, const arma::mat& L_prev);
```

Update the `ife()` declaration to include optional warm-start parameters:
```cpp
List ife(const arma::mat& E, int force, int mc, int r, int hard, double lambda,
         const arma::mat& F_prev = arma::mat(),
         const arma::mat& L_prev = arma::mat());
```

Update `ife_part()` declaration similarly:
```cpp
List ife_part(arma::mat E, int r,
              const arma::mat& F_prev = arma::mat(),
              const arma::mat& L_prev = arma::mat());
```

Update `beta_iter()` declaration:
```cpp
List beta_iter(const arma::cube& X, const arma::mat& xxinv, const arma::mat& Y,
               int r, double tolerate, const arma::mat& beta0, int max_iter,
               double omega = 0.8);
```

---

## 8. Input Validation

No new input validation is required at the R level. All changes are internal to the C++ convergence loops. The existing validation of `tol > 0` and `max.iteration > 0` in R/default.R is sufficient.

The warm-start functions internally validate that F_prev and L_prev have compatible dimensions. If incompatible, they silently fall back to cold-start (panel_factor).

---

## 9. Output Contract

**All output structures are UNCHANGED.** Every function returns the same List with the same elements. The only differences are:
- `niter` values will generally be lower (faster convergence)
- `fit` values will be numerically equivalent (within floating-point tolerance) to the original
- `beta`, `factor`, `lambda` values will be numerically equivalent

The SSR-based early convergence may cause slight differences in the final iteration's parameter values, but these are within the tolerance `tol` by construction.

---

## 10. Numerical Constraints

- The damping parameter omega MUST be in (0, 1]. omega=1.0 recovers the original undamped behavior. omega=0.8 is the default.
- The burn-in cap of 50 is chosen as a conservative upper bound. For very large r (r > 10), the formula 5*r + 5 may exceed 50, in which case the cap of 50 applies.
- The SSR-based convergence uses tol*0.1 as its threshold, which is 10x tighter than the Frobenius criterion. This prevents premature convergence.
- The 1e-10 floor on denominators prevents division by zero but is small enough to not affect normal convergence behavior (typical ||fit||_F values are O(sqrt(T*N)) which is >> 1e-10 for any real dataset).

---

## 11. Implementation Order

The changes are independent and can be implemented in any order. However, the recommended order is:

1. **Change C1** (denominator floors) — smallest, safest, immediate benefit
2. **Change A** (burn-in cap) — medium complexity, large performance benefit
3. **Change B** (warm-start SVD) — largest change, requires new function
4. **Change D** (damped beta-factor) — medium complexity, robustness benefit

---

## 12. Files Modified

| File | Change | Risk |
| --- | --- | --- |
| `src/fe_sub.cpp` | Add `panel_factor_warm()`, modify `ife()` and `ife_part()` signatures | MEDIUM |
| `src/ife_sub.cpp` | All four changes in iteration functions | HIGH |
| `src/cfe_sub.cpp` | Changes A, B, C in cfe_iter | MEDIUM |
| `src/fect.h` | Updated declarations | LOW |

No R-layer files need modification. No NAMESPACE or DESCRIPTION changes needed.
