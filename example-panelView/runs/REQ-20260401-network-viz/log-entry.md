<!-- filename: 2026-04-01-network-viz-type.md -->

# 2026-04-01 -- Add `type = "network"` to panelView

> Run: `REQ-20260401-network-viz` | Profile: r-package | Verdict: PASS

## What Changed

Added a new `type = "network"` option to `panelview()` that visualizes the bipartite graph structure of the panel's observation matrix, inspired by Correia (2016) Figure 2. Units and time periods become two distinct node sets (different shapes and colors), edges represent observations, singletons (degree-1 nodes) are highlighted in red, and connected components are enclosed in convex hulls. The feature extends to k-partite graphs when additional fixed-effect columns are provided via the new `fe` parameter. igraph is a conditional dependency (Suggests) used for graph computation, while ggplot2 handles all rendering.

## Files Changed

| File | Action | Description |
| --- | --- | --- |
| `R/plot-network.R` | created | New 265-line function `.pv_plot_network(s)` implementing bipartite/k-partite graph construction, singleton detection, component analysis, and ggplot2 visualization |
| `R/panelView.R` | modified | Added `fe = NULL` parameter, `"network"` to match.arg, `ignore.treat = TRUE` guard, formula bypass for network type, time-gap guard, `seq_along` fix, network dispatch branch |
| `DESCRIPTION` | modified | Added igraph to Suggests, rlang to Imports |
| `NAMESPACE` | modified | Added `importFrom("rlang", ".data")` |
| `man/panelview.Rd` | modified | Documented `type = "network"`, `fe` parameter, and network return value |
| `tests/testthat/test-network.R` | created | 78 test assertions covering all scenarios from test-spec.md |

## Process Record

### Proposal (from planner)

**Implementation spec summary** (from `spec.md`):
- Build bipartite graph from observation matrix I using `igraph::graph_from_biadjacency_matrix()` for k=2; general edge-list construction for k>2 (k-partite)
- Identify singletons via `igraph::degree() == 1`; compute connected components via `igraph::components()`
- Render with ggplot2 (not igraph's base plot): convex hulls around components, color/shape by FE dimension, red highlighting for singletons
- New `fe` parameter for k-partite extension; `ignore.treat = TRUE` to bypass formula requirements
- igraph in Suggests with `requireNamespace()` check; all calls namespace-qualified

**Test spec summary** (from `test-spec.md`):
- 6 main scenarios: balanced panel (K_{4,3}), unbalanced with 5 singletons and 3 components, no-formula call, return structure, turnout dataset, vertex attributes
- 6 edge cases: single unit, single time period, fully disconnected, igraph unavailability, invalid fe, minimal 2x2
- 2 k-partite scenarios: 3-way FE (unit x time x region), 4-way FE
- 8 property-based invariants: vertex count, edge count, bipartite property, singleton degree, component membership, component sizes sum, balanced no-singletons, balanced one-component
- All metrics exact (integer/boolean graph properties); no floating-point tolerances needed

### Implementation Notes (from builder)

- Used explicit variable extraction (`I <- s$I; N <- s$N; ...`) instead of `with(s, {...})` for cleaner igraph namespace-qualified calls
- Changed `1:length(varnames)` to `seq_along(varnames)` to handle empty varnames safely (network type sets `varnames <- character(0)`)
- Used `ignore.treat <- TRUE` (boolean) instead of `1` (numeric); both work with existing comparisons since `TRUE == 1` in R
- Extended data subsetting to `data[,c(index, Y, D, X, fe)]`; no-op when fe is NULL
- Network dispatch placed first in the if/else chain to short-circuit before treatment processing
- Used `ggplot2::aes(.data$x, ...)` with rlang .data pronoun for clean R CMD check

### Validation Results (from tester)

**Per-Test Result Table** (from `audit.md`):

| Test | Metric | Expected | Actual | Tolerance | Verdict |
| --- | --- | --- | --- | --- | --- |
| 2.1 balanced panel | vertex count | 7 | 7 | exact | PASS |
| 2.1 balanced panel | edge count | 12 | 12 | exact | PASS |
| 2.1 balanced panel | singleton count | 0 | 0 | exact | PASS |
| 2.1 balanced panel | component count | 1 | 1 | exact | PASS |
| 2.1 balanced panel | component size | 7 | 7 | exact | PASS |
| 2.1 balanced panel | min degree >= 2 | TRUE | TRUE | exact | PASS |
| 2.2 unbalanced panel | vertex count | 11 | 11 | exact | PASS |
| 2.2 unbalanced panel | edge count | 9 | 9 | exact | PASS |
| 2.2 unbalanced panel | singleton count | 5 | 5 | exact | PASS |
| 2.2 unbalanced panel | component count | 3 | 3 | exact | PASS |
| 2.2 unbalanced panel | component sizes (sorted) | c(6,3,2) | c(6,3,2) | exact | PASS |
| 2.3 no formula | no error | no error | no error | exact | PASS |
| 2.4 return structure | names | graph,singletons,components | graph,singletons,components | exact | PASS |
| 2.5 turnout data | vertex count = states+years | match | match | exact | PASS |
| 2.6 vertex attributes | fe_type distinct | unit,time | unit,time | exact | PASS |
| 3.1 single unit | singleton count | 3 | 3 | exact | PASS |
| 3.2 single time period | singleton count | 5 | 5 | exact | PASS |
| 3.3 fully disconnected | singleton count | 8 | 8 | exact | PASS |
| 3.3 fully disconnected | component count | 4 | 4 | exact | PASS |
| 3.4 igraph mock | error mentioning "igraph" | match | match | exact | PASS |
| 3.5 invalid fe (not char) | error | error | error | exact | PASS |
| 3.5 invalid fe (missing col) | error | error | error | exact | PASS |
| 3.5 invalid fe (dup index) | error | error | error | exact | PASS |
| 3.6 minimal 2x2 | 0 singletons, 1 component | match | match | exact | PASS |
| 4.1 three-way FE | fe_type distinct count | 3 | 3 | exact | PASS |
| 4.1 three-way FE | vertex count | 9 | 9 | exact | PASS |
| 4.2 four-way FE | fe_type distinct count | 4 | 4 | exact | PASS |
| 5.1-5.8 invariants | all 8 invariants | hold | hold | exact | PASS |

Summary: 78 test assertions executed, 78 passed, 0 failed.

**Before/After Comparison Table**:

| Metric | Before | After | Change | Interpretation |
| --- | --- | --- | --- | --- |
| Existing test suite (test-panelview.R) | 36/36 PASS | 36/36 PASS | 0 | No regression |
| R CMD check errors | 0 | 0 | 0 | Clean |
| R CMD check warnings | 0 | 1 (version gate) | +1 | Expected CRAN version gate; not a real issue |
| R CMD check notes | 0 | 1 (.git dir) | +1 | Worktree artifact; absent in release tarball |

Additional notes:
- 2 cosmetic ggplot2 scale warnings on fully disconnected panels (all nodes are singletons, so `scale_color_manual` has no non-singleton data). Does not affect correctness.
- R CMD check run with `--as-cran --no-manual`. All sections pass: install, dependencies, R code, S3 consistency, Rd, examples, tests.
- igraph namespace qualification verified: 20+ `igraph::` calls, no `library(igraph)` or `require(igraph)`.

### Problems Encountered and Resolutions

| # | Problem | Signal | Routed To | Resolution |
| --- | --- | --- | --- | --- |
| 1 | Single time period crash: `diff(sort(unique(time)))` returns empty vector when TT=1, causing `min()`/`max()` to fail with `Inf`/`-Inf`, then `if()` gets NA | BLOCK | builder | Builder wrapped the entire time-gap computation block (lines 362-417) in `if (type != "network") { ... }` guard. Network type does not need time-gap validation. |
| 2 | `.data` pronoun not imported: R CMD check NOTE about undefined global variable `.data` in plot-network.R | BLOCK | builder | Builder added rlang to Imports in DESCRIPTION and `importFrom("rlang", ".data")` to NAMESPACE. rlang is a transitive dependency via ggplot2, so no new install burden. |
| 3 | Test 3.4 (igraph unavailability) failed during R CMD check: test read source file via `system.file()` which does not exist for compiled R packages | (tester fix) | tester | Tester rewrote the test to use `testthat::local_mocked_bindings()` to mock `requireNamespace` returning FALSE for igraph, then verified the behavioral error message. Works in both `devtools::load_all()` and `R CMD check` contexts. |

### Review Summary

Pending -- reviewer review follows scriber.

- **Pipeline isolation**: pending
- **Convergence**: pending
- **Tolerance integrity**: pending
- **Verdict**: pending

## Design Decisions

1. **igraph for computation, ggplot2 for rendering**: igraph provides robust graph algorithms (bipartite construction, degree, components, layout) but its `plot.igraph()` uses base R graphics. Using ggplot2 for rendering maintains visual consistency with the rest of panelView and gives users the full ggplot2 customization API.

2. **`fe` parameter for k-partite extension**: A single character vector parameter (`fe`) specifies additional FE dimension column names beyond the standard unit and time from `index`. When `fe = NULL` (default), the standard bipartite graph is built. When `fe` is provided, a k-partite graph is constructed with k = 2 + length(fe) node types. This keeps the API minimal while supporting arbitrary FE dimensionality.

3. **Network dispatch first in if/else chain**: The network branch is checked before outcome/treat/bivariate because it needs to short-circuit -- network plots do not use `obs.missing`, treatment variables, or many other objects that later types require.

4. **Time-gap guard**: Rather than making the time-gap computation handle the single-period edge case, the entire block was bypassed with `if (type != "network")`. Network plots are purely structural (graph topology) and do not depend on time spacing, so this is the correct abstraction.

5. **`seq_along()` for empty vector safety**: The original `1:length(varnames)` produces `c(1, 0)` when varnames has length 0 (network type). `seq_along()` correctly returns an empty sequence. This is a defensive fix that does not change behavior for other types.

6. **Convex hull degenerate handling**: Components with fewer than 3 nodes skip hull drawing (only edges/points shown). Components with 3+ nodes use `grDevices::chull()` for the convex hull polygon.

7. **Vertex name collision in bipartite case**: Unit and time labels are not prefixed in the k=2 case (a unit named "3" and period "3" share the same `V(g)$name`). The `fe_type` attribute distinguishes them. Prefixing was considered but rejected to keep labels readable. The k-partite case uses prefixed names to avoid ambiguity across 3+ dimensions.

## Handoff Notes

- The `fe` parameter is passed through the environment list `s` like all other panelview parameters. Any new parameters added to `panelview()` will automatically be available to `.pv_plot_network()`.
- The ggplot2 color scale warning on fully disconnected panels (test 3.3) is cosmetic and harmless. Silencing it would require adding a conditional `scale_color_manual` only when non-singleton nodes exist, which adds complexity for no functional benefit.
- The observation matrix `I` is the biadjacency matrix. For very large panels, `display.all = FALSE` (the default) samples 500 units before I is constructed, so the graph is built from the sampled panel. This is consistent with other panelView types but may surprise users expecting the full graph.
- Non-unique vertex names in the bipartite case are a known limitation. If downstream code compares vertices by name, it must also check `fe_type`. The k-partite case avoids this with prefixed names.
- The test for igraph unavailability (test 3.4) uses `testthat::local_mocked_bindings()` to mock `requireNamespace`. This is fragile if testthat changes its mocking API. A more robust alternative would be to extract the igraph check into a separate internal function that can be easily mocked.
