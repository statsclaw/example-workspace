# Mailbox

## Handoff Summary (from planner)

**Date**: 2026-03-31

### For Builder (spec.md)

Builder should implement four independent changes to the C++ convergence core: (A) cap the burn-in factor count at min(5*r+5, d, 50) in fe_ad_inter_iter, fe_ad_inter_covar_iter, and cfe_iter; (B) add a warm-start SVD function panel_factor_warm() and integrate it into beta_iter, ife, ife_part, and the iteration loops; (C) add 1e-10 denominator floors to all convergence metrics in fe_ad_iter, fe_ad_covar_iter, and cfe_iter, plus an SSR-based early convergence check; (D) add damped beta-factor alternation in beta_iter with omega=0.8 default and joint convergence monitoring. Files to modify: src/fe_sub.cpp (new function + ife/ife_part signature changes), src/ife_sub.cpp (all four changes), src/cfe_sub.cpp (changes A, B, C), src/fect.h (updated declarations). No R-layer changes needed.

### For Tester (test-spec.md)

Tester should validate: (1) R CMD check passes clean, (2) all existing tests pass, (3) accuracy preserved across FE/IFE/CFE estimators using 11 test scenarios with specific seeds and expected bounds, (4) convergence speed improved on moderate (N=200) and large (N=500) panels, (5) robustness improved across seeds and tolerance levels, (6) edge cases (r=0, small panel, near-zero values, max.iteration=1) handled correctly. The test-spec provides complete R code for all scenarios with exact acceptance thresholds.
