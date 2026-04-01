# Active Handoff — example-fect

> Last updated: 2026-03-31 from run `convergence-improvements`

- The `panel_factor_warm()` function is the largest new addition. If future changes modify the SVD logic in `panel_factor()`, the warm-start variant must be kept in sync.
- The Rcpp-generated R wrappers for `ife()` and `ife_part()` show warnings about unparseable default arguments (`arma::mat()`). These are harmless because these functions are only called from C++ internally, never from R. However, if someone adds R-level calls to these functions in the future, they will need to pass the warm-start arguments explicitly.
- The burn-in cap of 50 is conservative. For very large factor models (r > 10), the formula `5*r+5` may exceed 50 and get clipped. If users report slow convergence with large r values, consider raising this ceiling.
- The damping parameter omega=0.8 is not exposed to the R level. If users need to tune it, a future change could add it as a parameter to `fect()`. For now, the default works well across all tested scenarios.
- The SSR-based convergence check uses `FE_adj()` from auxiliary.cpp. If the definition of residuals changes, the SSR computation must be updated accordingly.
- Niter values will generally be lower after these changes. Any regression tests that assert exact iteration counts may need updated baselines.
