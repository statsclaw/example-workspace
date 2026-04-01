# Impact Analysis

## Issue

GitHub issue #1: "Convergence performance and robustness improvements"

Two sub-problems:
1. **Convergence speed** — EM/optimization loop converges slowly on moderate-to-large panels (N > 200, T > 30)
2. **Robustness** — Estimates sensitive to initial conditions and tuning parameters

## Affected Files

### Primary — C++ convergence core (HIGH RISK)

| File | Functions | Risk | Reason |
| --- | --- | --- | --- |
| `src/ife_sub.cpp` | `fe_ad_iter()`, `fe_ad_covar_iter()`, `fe_ad_inter_iter()`, `fe_ad_inter_covar_iter()`, `beta_iter()` | HIGH | Core EM iteration loops — convergence logic, tolerance checks, burn-in |
| `src/cfe_sub.cpp` | `cfe_iter()` | HIGH | Complex FE iteration — multi-component convergence |
| `src/ife.cpp` | `inter_fe()`, `inter_fe_ub()` | MEDIUM | Dispatch to sub-routines; initialization logic |
| `src/fe_sub.cpp` | `panel_factor()`, `panel_beta()` | MEDIUM | SVD factor estimation, beta estimation — called every iteration |
| `src/auxiliary.cpp` | `E_adj()`, `wE_adj()` | MEDIUM | EM E-step helpers |

### Secondary — R layer

| File | Functions | Risk | Reason |
| --- | --- | --- | --- |
| `R/fe.R` | `fect_fe()` | LOW | Calls C++ with tolerance/max_iter params; initial condition setup |
| `R/cfe.R` | `fect_cfe()` | LOW | Calls C++ CFE |
| `R/default.R` | defaults | LOW | Default tol=1e-3, max.iteration=1000 |

### Header

| File | Risk | Reason |
| --- | --- | --- |
| `src/fect.h` | LOW | Declarations — may need signature updates |

## Key Optimization Bottlenecks Identified

1. **Burn-in phase** (ife_sub.cpp:287-295, cfe_sub.cpp:473-481): When weights are used, inflated r_burnin = d - niter forces two full convergence passes. Slow on large panels.
2. **SVD every iteration** (fe_sub.cpp `panel_factor()`): O(TNr) per iteration for factor extraction. No warm-starting from previous SVD.
3. **Sequential multi-component updates** (cfe_sub.cpp:439-531): CFE updates 5 components one-by-one each iteration.
4. **Naive convergence metric**: Frobenius norm of relative change can be slow to satisfy; small denominator issues near convergence.
5. **OLS initialization**: LSDV starting point may be poor with many unobserved factors, requiring many extra iterations.

## Robustness Issues

1. **Initial condition sensitivity**: OLS/LSDV initialization (fe.R:88-96) can produce poor starting values when true factor structure is complex.
2. **No warm-starting of SVD**: Each iteration recomputes full SVD from scratch.
3. **Beta-factor oscillation**: `beta_iter()` can oscillate when r is large relative to data.

## Required Teammates

| Teammate | Purpose |
| --- | --- |
| planner | Analyze convergence algorithms, design acceleration strategy (e.g., Anderson acceleration, better initialization, warm-start SVD), produce spec.md and test-spec.md |
| builder | Implement C++ convergence improvements in worktree |
| tester | Validate R CMD check, existing tests pass, convergence improvements measurable |
| scriber | Record architecture, process log, documentation |
| reviewer | Cross-check pipeline isolation, verify no regression |
| shipper | Push to dev branch, create PR to master, comment on issue #1 |

## Profile

r-package

## Write Surface (Builder)

- `src/ife_sub.cpp` — convergence loop improvements
- `src/cfe_sub.cpp` — CFE convergence improvements
- `src/ife.cpp` — initialization improvements
- `src/fe_sub.cpp` — warm-start SVD, panel_factor improvements
- `src/auxiliary.cpp` — E-step optimizations (if needed)
- `src/fect.h` — signature updates (if needed)
- `R/fe.R` — initialization logic improvements (if needed)
- `R/cfe.R` — parameter passing (if needed)
- `R/default.R` — default parameter adjustments (if needed)

## Simplified Workflow Evaluation

**NOT eligible** — algorithmic changes to numerical methods, affects convergence behavior, touches many files, high risk of regression. Full workflow required.
