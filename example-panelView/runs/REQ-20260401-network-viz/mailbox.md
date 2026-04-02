# Mailbox — REQ-20260401-network-viz

## Handoff: planner -> builder (2026-04-01)

Builder should implement the `type = "network"` feature per `spec.md`. Key sections: Section 2 (API Surface) for the three changes to `panelView.R` (match.arg, ignore.treat guard, dispatch block) and the new `fe` parameter; Section 3 (Algorithm Steps) for the full `.pv_plot_network()` implementation in `R/plot-network.R`; Section 7 (Implementation Notes) for R-package conventions (igraph in Suggests, namespace-qualified calls, DESCRIPTION update, .Rd documentation). The observation matrix `I` is already available in the environment list `s`. Edge cases are in Section 8.

## Handoff: planner -> tester (2026-04-01)

Tester should validate per `test-spec.md`. Key sections: Section 2 (Test Scenarios) has 6 concrete scenarios with exact expected values for graph structure, singletons, and components; Section 3 (Edge Cases) covers degenerate panels and invalid inputs; Section 4 (k-partite) tests the multi-FE extension; Section 5 (Invariants) lists 8 property-based checks that must hold for ANY valid input. Tests go in `tests/testthat/test-network.R` following the existing helper pattern. R CMD check must pass with 0 errors, 0 warnings.

## Interface Changes: builder (2026-04-01)

1. **New parameter `fe`** added to `panelview()` signature (after `report.missing`, default NULL). Character vector of additional FE column names for k-partite graphs.
2. **New `type = "network"`** option in `match.arg()`. Does not require formula/Y/D.
3. **Return value**: When `type = "network"`, returns invisible list with `graph` (igraph object), `singletons` (data.frame), `components` (data.frame). Other types unaffected.
4. **New internal function `.pv_plot_network(s)`** in `R/plot-network.R` (not exported).
5. **Data subsetting** extended to `data[,c(index, Y, D, X, fe)]` to preserve fe columns. No-op when fe is NULL.
6. **Variable name loop** changed from `1:length(varnames)` to `seq_along(varnames)` for empty vector safety.

## BLOCK: tester -> leader (2026-04-01)

**Signal**: BLOCK
**Route to**: builder

Two issues require builder fixes:

1. **Critical -- single time period crash**: When a panel has only 1 unique time period, `panelView.R` crashes at the time gap computation (`diff(sort(unique(time)))` returns empty vector, `min()`/`max()` get no arguments, `if()` receives NA). The `type == "network"` path does not skip this code. Builder must guard the time gap computation so network type bypasses it, or handle the 1-unique-time edge case.

2. **R CMD check NOTE -- `.data` not imported**: `plot-network.R` uses `ggplot2::aes(.data$x, ...)` but `.data` is not in the NAMESPACE. Builder needs to add `@importFrom rlang .data` or equivalent, and add `rlang` to Imports in DESCRIPTION if not already present.

After fixes, re-dispatch tester for re-validation.
