# Comprehension Record

## Request ID
REQ-20260401-network-viz

## Date
2026-04-01

## Input Materials Read

| Material | Type | Content |
|----------|------|---------|
| `request.md` | Run artifact | Scope, acceptance criteria for `type = "network"` |
| `impact.md` | Run artifact | Affected files, risk areas, required teammates |
| `correia2016-notes.md` | Reference notes | Key concepts from Correia (2016) on bipartite graphs of FE |
| `R/panelView.R` | Target repo source | Main function: signature (lines 8-55), type matching (line 109), dispatch block (lines 1150-1161), observation matrix `I` construction (lines 625-692) |
| `R/plot-treat.R` | Target repo source | Example of `.pv_plot_*()` pattern: receives `s` (environment list), uses `with(s, {...})` |
| `DESCRIPTION` | Target repo metadata | Current Suggests: testthat only. igraph to be added. |
| `NAMESPACE` | Target repo metadata | Exports only `panelview()`. Uses `importFrom` for ggplot2, gridExtra, grid, dplyr. |

## Restated Core Requirement

Add a `type = "network"` option to `panelview()` that builds and visualizes a bipartite graph from the panel data's observation matrix. Units and time periods become two distinct sets of nodes (different shapes and colors). An edge connects unit i to time t when unit i is observed at time t. The visualization highlights singletons (degree-1 nodes: units observed in only one period, or periods with only one unit) and draws convex hulls around connected components to show which groups of units and periods are identifiable together. The feature extends to k-partite graphs when 2+ fixed-effect dimensions are provided. igraph is a soft dependency (Suggests). The function returns an invisible list containing the igraph object, a singletons data frame, and component membership info.

## Key Concepts Verified

### Bipartite Graph of Fixed Effects (Correia 2016, Figure 1-2)

A bipartite graph G = (V_unit UNION V_time, E) where:
- V_unit = {u_1, ..., u_N}: one node per unit
- V_time = {t_1, ..., t_T}: one node per time period
- Edge (u_i, t_j) exists iff unit i is observed at time j, i.e., I[j, i] = 1

The observation matrix `I` (TT x N, binary) already computed in `panelView.R` is exactly the biadjacency matrix (incidence matrix) of this bipartite graph. `I[j, i] = 1` means row j (time period) and column i (unit) are connected.

### Singletons (Correia 2016, Section 3.4)

A singleton is a degree-1 node in the bipartite graph:
- A unit node u_i with degree 1: unit i is observed in exactly one time period (colSums(I)[i] == 1)
- A time node t_j with degree 1: period j has exactly one observed unit (rowSums(I)[j] == 1)

These correspond to fixed-effect categories that absorb all variation from a single observation, making them problematic for identification. In a balanced panel there are no singletons (every node has degree >= 2).

### Connected Components (Correia 2016, Section 3.5)

The bipartite graph may decompose into disconnected subgraphs (connected components). Fixed effects are only identified within a connected component -- units in different components cannot be compared. igraph provides `igraph::components()` for this.

### k-partite Extension (Correia 2016, Section 4.1)

When f > 2 fixed-effect dimensions exist (e.g., unit x time x region), the bipartite graph generalizes to a k-partite graph. Each fixed-effect dimension defines a node type. An observation creates edges between ALL pairs of FE categories it belongs to (or, more simply, a hyperedge connecting one node from each dimension). For practical visualization, we construct a standard graph where each observation contributes edges between each pair of its FE category nodes. igraph's `bipartite` functions only handle 2 types; for k > 2 we construct a general graph with a `type` vertex attribute taking k distinct values.

## Formulas Verified

1. **Biadjacency matrix**: I is TT x N, binary. I[j, i] = 1 iff unit i observed at time j.
2. **Degree of unit node i**: deg(u_i) = sum_j I[j, i] = colSums(I)[i]
3. **Degree of time node j**: deg(t_j) = sum_i I[j, i] = rowSums(I)[j]
4. **Singleton condition**: deg(v) == 1 for any node v
5. **Edge weight** (for k-partite): number of observations sharing a pair of FE categories. For standard bipartite with no repeated measures, weight = 1.

## Questions and Gaps

None. All concepts are well-defined from the reference material and the codebase.

## Design Decisions (Planner's Judgment)

1. **Rendering approach**: The package is ggplot2-based. igraph has its own `plot.igraph()` but it uses base R graphics. For consistency, the spec will use igraph for graph computation (layout, components, degrees) and ggplot2 for rendering (points, segments, polygons). This keeps the visual style consistent with the rest of panelView.

2. **Layout algorithm**: `igraph::layout_as_bipartite()` for k=2. For k>2, `igraph::layout_with_fr()` (Fruchterman-Reingold) which handles general graphs well.

3. **New parameter name for extra FE dimensions**: `fe` (a character vector of column names). When `fe` is provided alongside `index`, the graph is k-partite with k = 2 + length(fe). When `fe = NULL` (default), standard bipartite (unit x time).

4. **Convex hull for degenerate components**: Components with 1 node get a point highlight only (no hull). Components with 2 nodes get a line segment or enlarged highlight. Components with 3+ nodes get `grDevices::chull()`.

5. **`type = "network"` should bypass formula/treatment requirements**: The network view only needs the panel structure (index columns), not a formula, outcome, or treatment. `ignore.treat` will be forced to TRUE internally.

6. **Early dispatch**: The `type = "network"` dispatch should occur early in panelView(), after basic data validation and `I` matrix construction, but before treatment-specific processing that is irrelevant for network plots.

## Comprehension Verdict

**FULLY UNDERSTOOD**

## HOLD Rounds Used
0

