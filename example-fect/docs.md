# Documentation Summary — REQ-20260401-104739

## What Changed and Why

The fect package's convergence speed and robustness were improved in response to Issue #1. Four backward-compatible changes were made to the EM-based estimation pipeline for interactive fixed effects models. These changes affect the R-level interface (one new parameter) and internal C++ return values (one new field), with no breaking changes to existing behavior.

## New Parameter: `n.init`

**Added to**: `fect()` function signature (in `R/default.R`)

**Type**: Positive integer (default: 1)

**Purpose**: Controls the number of random initializations for the EM algorithm. When `n.init = 1` (default), behavior is identical to before -- a single deterministic starting point from `initialFit()`. When `n.init > 1`, the function generates perturbed copies of the initial fit, runs trial estimations for each, and selects the initialization that achieves the lowest residual variance (`sigma2`). This improves robustness to local optima.

**Usage example**:
```r
out <- fect(Y ~ D + X1 + X2, data = simdata, index = c("id", "time"),
            method = "ife", r = 2, CV = FALSE, se = FALSE,
            n.init = 3)
```

**Documentation status**: `man/fect.Rd` has been updated with:
- `n.init = 1` in the `\usage` section (between `max.iteration` and `seed`)
- `\item{n.init}{...}` entry in the `\arguments` section

## New Diagnostics: Convergence Warnings

When the EM algorithm reaches `max.iteration` without achieving convergence (i.e., the relative Frobenius norm change `dif` remains above `tol`), the package now emits an R-level `warning()`:

```
EM algorithm did not converge within 1000 iterations. Consider increasing max.iteration or relaxing tol.
```

This warning is emitted only for the final estimation call, not for CV inner-loop calls (where non-convergence at the loose `cv_tol` is expected and acceptable).

The warning is informational only -- estimation still returns results. It helps users identify cases where results may be imprecise due to incomplete convergence.

## Internal Changes (No User-Facing Doc Required)

1. **Warm-start CV**: Cross-validation now reuses fitted values from the previous `r` candidate (IFE) or `lambda` candidate (MC) as the starting point for the next. This is entirely internal and transparent to users. No parameter controls it -- it is always active.

2. **Burn-in fix**: In weighted IFE estimation, the burn-in phase's converged solution is now preserved as the warm start for the real estimation phase, rather than being discarded. This is internal to the C++ backend.

3. **`converged` field**: All C++ iteration functions now include a `converged` integer (1 or 0) in their return lists. This is an additive, non-breaking change to internal return structures. It is not exposed to users directly but drives the warning mechanism above.

4. **`perturbedFit()`**: New internal (non-exported) R function in `R/support.R`. Generates Gaussian-perturbed copies of initial values for the multi-start mechanism. Not user-callable.

## Impact on Existing Users

- **Default behavior unchanged**: With `n.init = 1` (default), all results are numerically identical to the previous version. Verified: ATT, sigma2, and coefficient estimates match exactly (tolerance 1e-12).
- **No API breaks**: The `fect()` return structure is unchanged. The `n.init` parameter has a default value, so existing call sites work without modification.
- **New warnings may appear**: Users who were silently hitting `max.iteration` without convergence will now see a warning. This is informative, not a behavior change -- their results were already potentially imprecise.

## Doc Files Modified

| File | Action | Description |
| --- | --- | --- |
| `man/fect.Rd` | modified | Added `n.init = 1` parameter documentation |
| `NAMESPACE` | modified | Added `rnorm` to stats imports (for `perturbedFit()`) |
| `ARCHITECTURE.md` | modified | Updated system architecture with convergence improvement patterns, highlighted changed modules/functions |

## Doc Generation Commands

No additional `devtools::document()` run is needed -- the `man/fect.Rd` was edited directly (not via roxygen2). If roxygen2 is used in the future, ensure `@param n.init` is added to the roxygen block in `R/default.R`.

## Deferred Items

- Vignette coverage: The convergence improvements could benefit from a brief section in the IFE vignette explaining when to use `n.init > 1`. Deferred to a future docs-only run.
- `perturbedFit()` could be documented with roxygen2 `@keywords internal` for developer reference. Not required for CRAN.

## ARCHITECTURE.md

Confirmed: `ARCHITECTURE.md` was produced and written to both the target repo root and the run directory. It includes updated module structure, function call graph, and data flow diagrams with all changed modules/functions highlighted in blue. New architectural patterns documented: warm-start CV, multi-start initialization, burn-in warm-start, convergence diagnostics.
