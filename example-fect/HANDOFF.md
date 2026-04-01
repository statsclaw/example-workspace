# Active Handoff — example-fect

> Last updated: 2026-04-01 from run `convergence-robustness-fix`

- The `n.init` parameter currently applies only at the initialization phase before CV, not within each CV fold. If robustness issues persist within individual folds, a per-fold multi-start could be considered (at significant computational cost).
- The `converged` flag is currently checked only for the final estimation (not CV inner loops). If detailed per-fold convergence tracking is needed, the flag is already available in the `inter_fe_ub()` return list.
- `RcppExports.cpp` and `R/RcppExports.R` were NOT regenerated. The `converged` field additions are backward-compatible (additive to return lists), but a `Rcpp::compileAttributes()` run before CRAN submission would be good practice.
- The warm-start caches (`warm_fit_cv`, `warm_fit_full`) are R lists held in memory during the CV loop. For very large panels (T x N > 10^6), this doubles memory usage during CV. If memory becomes an issue, the cache could be limited to the most recent candidate only.
- All remaining `R CMD check` findings (pdflatex, tidy, V8, CRAN version) are environment-specific and unrelated to the convergence changes.
