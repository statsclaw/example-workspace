# Request: Add `type = "network"` to panelView

## Request ID
REQ-20260401-network-viz

## Target Repository
- Name: example-panelView
- URL: https://github.com/statsclaw/example-panelView.git
- Checkout: /Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/example-panelView
- Branch: main

## Workspace Repository
- URL: https://github.com/TianzhuQin/workspace
- Checkout: /Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/workspace

## Scope

Add a new `type = "network"` option to the `panelview()` function that visualizes the bipartite graph structure of the panel's observation matrix, inspired by Correia (2016) "A Feasible Estimator for Linear Models with Multi-Way Fixed Effects" (Figure 2).

### Feature Requirements

1. **Bipartite graph construction**: Build a graph from the observation matrix where units and time periods are differently shaped/colored nodes, and edges represent "unit observed in period"
2. **Singleton detection**: Identify and highlight degree-1 nodes (units observed in only one period, or periods with only one unit) — these correspond to the singletons discussed in Correia (2016) Section 3.4
3. **Connected component visualization**: Draw convex hulls around connected components
4. **k-partite extension**: Support 2+ sets of fixed effects (not just unit × time, but e.g., unit × time × region)
5. **Dependency management**: igraph in Suggests, not Imports
6. **Return value**: igraph object, singletons dataframe, and component info (invisibly, as a list)

### Reference Material
- Correia (2016) — bipartite graph of fixed effects (Figure 1 & 2), graph pruning of degree-1 nodes (Section 3.4), connected components (Section 3.5), k-partite extension for f>2 (Section 4.1)
- fect package datasets: `sim_region` (unbalanced, ideal for demonstrating sparse networks), `gs2020` (treatment reversals)

## Acceptance Criteria

1. `panelview(data, formula, index, type = "network")` produces a bipartite network plot
2. Singletons (degree-1 nodes) are visually highlighted
3. Connected components have convex hulls drawn around them
4. k-partite graphs supported when additional fixed effect columns provided
5. igraph is in Suggests only; graceful error if not installed
6. Function returns (invisibly) a list with `graph` (igraph object), `singletons` (data.frame), `components` (component info)
7. R CMD check passes with zero errors, zero warnings, zero notes (excluding igraph availability note)
8. Works with both balanced and unbalanced panel data

## Workflow
Code Change (Workflow 1)
