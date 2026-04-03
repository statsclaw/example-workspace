# Impact Analysis

## Request ID
REQ-20260401-network-viz

## Profile
r-package

## Affected Files

### Modified Files

| File | Change Type | Risk |
|------|------------|------|
| `R/panelView.R` | Modify `type` argument match.arg + dispatch block | LOW — additive change to existing routing |
| `DESCRIPTION` | Add igraph to Suggests | LOW — metadata only |
| `NAMESPACE` | No change needed — igraph is Suggests, loaded at runtime | NONE |
| `man/panelview.Rd` | Add `type = "network"` documentation, new parameters | LOW |

### New Files

| File | Purpose | Risk |
|------|---------|------|
| `R/plot-network.R` | New `.pv_plot_network()` function — bipartite graph construction, singleton detection, component analysis, visualization | HIGH — new algorithmic code, igraph dependency |

## Risk Areas

1. **igraph conditional loading**: Must use `requireNamespace("igraph", quietly = TRUE)` and fail gracefully. All igraph calls must be namespace-qualified (`igraph::graph.incidence()`, etc.)
2. **Return value change**: `panelview()` currently returns a ggplot object. The `type = "network"` path must return a list (graph, singletons, components) invisibly — this is a new return contract for this type only. Other types are unaffected.
3. **k-partite extension**: Supporting >2 fixed effects means the bipartite graph generalizes to k-partite. igraph's bipartite functions only handle 2 modes. Custom graph construction needed for k>2.
4. **Convex hull computation**: Requires coordinate extraction from igraph layout + `grDevices::chull()`. Must handle degenerate cases (components with 1-2 nodes).
5. **Parameter namespace**: New parameters (e.g., `fe` for additional fixed effects columns) must not conflict with existing 50+ parameters.

## Required Teammates

| Teammate | Reason |
|----------|--------|
| planner | Paper comprehension (Correia 2016), spec for bipartite graph construction + singleton detection algorithm, test spec |
| builder | Implement `plot-network.R` + modify `panelView.R` routing |
| tester | Validate with balanced/unbalanced panels, singleton detection, component detection, k-partite |
| scriber | ARCHITECTURE.md update, process record |
| reviewer | Cross-compare pipeline outputs, verify graph-theoretic correctness |

## Workflow
Workflow 1 (Code Change): `leader → planner → builder → tester → scriber → reviewer`

## Write Surface

- `R/plot-network.R` (new)
- `R/panelView.R` (lines 109, 1150-1161 — type matching + dispatch)
- `DESCRIPTION` (Suggests field)
- `man/panelview.Rd` (documentation)
