# Implementation Specification: `type = "network"`

## Request ID
REQ-20260401-network-viz

## Profile
r-package

---

## 1. Notation

| Symbol | Type | Dimensions | Description |
|--------|------|-----------|-------------|
| N | integer scalar | 1 | Number of units (after filtering by `id`/`show.id`) |
| TT | integer scalar | 1 | Number of unique time periods |
| I | binary matrix | TT x N | Observation matrix. I[j, i] = 1 iff unit i observed at time j. Already computed in panelView.R. |
| V_unit | node set | N nodes | Unit nodes in the bipartite graph |
| V_time | node set | TT nodes | Time period nodes in the bipartite graph |
| E | edge set | |E| <= N*TT | Edges: (u_i, t_j) exists iff I[j, i] = 1 |
| deg(v) | integer scalar | 1 | Degree of node v: number of edges incident to v |
| fe | character vector or NULL | length k-2 or NULL | Additional FE column names (beyond unit and time from `index`) |
| k | integer scalar | 1 | Number of FE dimensions: k = 2 + length(fe). Default k = 2. |

---

## 2. API Surface

### 2.1 Changes to `panelview()` signature

Add to `match.arg()` on line 109: include `"network"` in the vector of valid types.

Add one new parameter to the function signature:

```
fe = NULL    # character vector of additional FE column names for k-partite graph (k > 2)
```

Place `fe` after `report.missing` in the argument list.

### 2.2 Changes to dispatch block (lines 1150-1161)

Insert a new branch for `type == "network"` in the dispatch block. The network dispatch should be checked FIRST (before "outcome", "treat", "bivariate") because it needs to short-circuit: network plots do not use `obs.missing`, treatment variables, or many other objects that later types require.

However, since the `I` matrix is constructed before the dispatch block (lines 625-692), and `s <- as.list(environment())` captures everything at line 1153, the `I` matrix will be available in `s`. The network branch can safely ignore treatment-related objects.

**Modified dispatch block:**

```r
s <- as.list(environment())
if (type == "network") {
    .pv_plot_network(s)
} else if (type == "outcome") {
    .pv_plot_outcome(s)
} else if (type == "treat") {
    .pv_plot_treat(s)
} else if (type == "bivariate") {
    .pv_plot_bivariate(s)
}
```

### 2.3 Type matching update

On line 109, change:

```r
type <- match.arg(type, c("treat", "missing", "miss", "outcome", "raw", "bivariate", "bivar"))
```

to:

```r
type <- match.arg(type, c("treat", "missing", "miss", "outcome", "raw", "bivariate", "bivar", "network"))
```

### 2.4 Formula relaxation for network type

The network type does not require a formula, outcome variable, or treatment variable. After the `type` is determined, if `type == "network"`, the function should:

1. Set `ignore.treat <- TRUE` (force — network plots do not use treatment)
2. Allow `formula = NULL`, `Y = NULL`, `D = NULL` without error

The existing code has validation logic that errors when formula/Y/D are missing. Add a guard early (after `type` is matched, around line 113):

```r
if (type == "network") {
    ignore.treat <- TRUE
}
```

This leverages the existing `ignore.treat` pathway which already skips Y/D/formula requirements (the code at lines 130-290 uses `ignore.treat` to bypass treatment parsing).

### 2.5 New function: `.pv_plot_network(s)`

File: `R/plot-network.R` (new file)

Signature: `.pv_plot_network <- function(s)`

Return: invisible list with components `graph`, `singletons`, `components` (see Section 5).

---

## 3. Algorithm Steps

### Step 1: Check igraph availability

```r
if (!requireNamespace("igraph", quietly = TRUE)) {
    stop("Package 'igraph' is required for type = 'network'. ",
         "Install it with: install.packages('igraph')",
         call. = FALSE)
}
```

### Step 2: Extract data from environment list `s`

From `s`, extract:
- `I`: the TT x N observation matrix (binary). If `I` is NULL (can happen if `leave.gap == 0` and data is already balanced with `dim(data)[1] == TT*N`), construct it as `matrix(1, TT, N)`.
- `N`, `TT`: dimensions
- `input.id`: unit labels (length N character/numeric vector)
- `raw.time`: time period labels (length TT)
- `index`: the c(unit_name, time_name) character vector
- `fe`: the additional FE column names (NULL for bipartite)
- `data`: the original data frame (needed for k-partite extension)
- Plot parameters: `main`, `xlab`, `ylab`, `cex.main`, `cex.axis`, `cex.lab`, `cex.legend`, `legendOff`, `theme.bw`, `color`

### Step 3: Build the graph

#### 3a. Bipartite case (k = 2, fe is NULL or length 0)

Use `igraph::graph_from_biadjacency_matrix()` (formerly `graph.incidence()`):

```r
g <- igraph::graph_from_biadjacency_matrix(I, directed = FALSE, weighted = TRUE)
```

This creates a bipartite graph where the first TT vertices are "time" type (rows of I) and the next N vertices are "unit" type (columns of I). igraph marks `V(g)$type` as FALSE for the first set (time) and TRUE for the second set (unit).

Set vertex attributes:
- `V(g)$name`: first TT nodes get `as.character(raw.time)`, next N nodes get `as.character(input.id)`
- `V(g)$fe_type`: first TT nodes get `index[2]` (time dimension label), next N nodes get `index[1]` (unit dimension label)
- `V(g)$fe_index`: integer index within each FE dimension (1:TT for time, 1:N for unit)

#### 3b. k-partite case (k > 2, fe is a non-empty character vector)

Build a general graph. Each observation (row in `data`) contributes edges between all pairs of FE categories it belongs to.

1. Collect all FE dimensions: `fe_dims <- c(index[1], index[2], fe)` -- length k
2. For each FE dimension d, extract unique categories: `cats_d <- unique(data[, d])`
3. Create vertex set: for each dimension d, one node per unique category. Total vertices = sum of unique categories across all dimensions.
4. Assign vertex attributes:
   - `name`: the category label (prefixed with dimension name to avoid collision: `paste0(d, ":", cat)`)
   - `fe_type`: the dimension name (d)
   - `type_id`: integer 1..k indicating which FE dimension
5. For each observation (row in data), for each pair of FE dimensions (d1, d2) where d1 < d2, add an edge between the node for `data[row, d1]` and the node for `data[row, d2]`. Use an edge list accumulated in a matrix, then create the graph.
6. Collapse parallel edges into weighted edges: `igraph::simplify(g, edge.attr.comb = list(weight = "sum"))`.

### Step 4: Compute degrees and identify singletons

```r
deg <- igraph::degree(g)
is_singleton <- (deg == 1)
singleton_nodes <- which(is_singleton)
```

Build the singletons data frame:

```r
singletons_df <- data.frame(
    node = igraph::V(g)$name[singleton_nodes],
    fe_type = igraph::V(g)$fe_type[singleton_nodes],
    degree = deg[singleton_nodes],
    stringsAsFactors = FALSE
)
```

If no singletons exist, `singletons_df` has 0 rows (same columns).

### Step 5: Compute connected components

```r
comp <- igraph::components(g)
```

`comp` is a list with:
- `membership`: integer vector (length = number of vertices), component ID for each vertex
- `csize`: integer vector, size of each component
- `no`: integer, number of components

Build component info:

```r
comp_info <- data.frame(
    component = seq_len(comp$no),
    size = comp$csize,
    stringsAsFactors = FALSE
)
```

Add component membership to vertex attributes:

```r
igraph::V(g)$component <- comp$membership
```

### Step 6: Compute layout

#### 6a. Bipartite (k = 2)

Use igraph's bipartite layout:

```r
lay <- igraph::layout_as_bipartite(g)
```

This produces a 2-column matrix. Row ordering matches vertex ordering. The two FE types are placed on two horizontal lines (y = 0 and y = 1). Swap so that time is on top (y = 1) and units on bottom (y = 0) if igraph assigns them the other way. Check `V(g)$type[1]` — if FALSE corresponds to time (first TT nodes), time gets y = 0 by default; swap by setting `lay[, 2] <- 1 - lay[, 2]` so time is on top.

Scale x-coordinates to [0, 1] within each type to spread nodes evenly.

#### 6b. k-partite (k > 2)

Use Fruchterman-Reingold:

```r
lay <- igraph::layout_with_fr(g)
```

### Step 7: Build ggplot2 visualization

Convert igraph layout + attributes into data frames suitable for ggplot2.

#### 7a. Node data frame

```r
node_df <- data.frame(
    x = lay[, 1],
    y = lay[, 2],
    name = igraph::V(g)$name,
    fe_type = igraph::V(g)$fe_type,
    is_singleton = is_singleton,
    component = comp$membership,
    stringsAsFactors = FALSE
)
```

#### 7b. Edge data frame

Extract edge list from igraph:

```r
el <- igraph::as_edgelist(g, names = FALSE)  # numeric indices
edge_df <- data.frame(
    x = lay[el[, 1], 1],
    y = lay[el[, 1], 2],
    xend = lay[el[, 2], 1],
    yend = lay[el[, 2], 2]
)
```

#### 7c. Convex hull data frame

For each connected component with 3+ nodes, compute convex hull polygon:

```r
hull_list <- list()
for (cid in seq_len(comp$no)) {
    idx <- which(comp$membership == cid)
    if (length(idx) >= 3) {
        pts <- lay[idx, , drop = FALSE]
        h <- grDevices::chull(pts)
        hull_list[[length(hull_list) + 1]] <- data.frame(
            x = pts[h, 1], y = pts[h, 2], component = cid
        )
    }
}
if (length(hull_list) > 0) {
    hull_df <- do.call(rbind, hull_list)
} else {
    hull_df <- NULL
}
```

#### 7d. Assemble ggplot

Build the plot layer by layer:

1. **Base**: `ggplot()` with `theme_bw()` (if `theme.bw` is TRUE) or `theme_minimal()`
2. **Convex hulls** (if `hull_df` is not NULL): `geom_polygon(data = hull_df, aes(x, y, group = component), fill = NA, color = "grey60", linetype = "dashed", linewidth = 0.5)`
3. **Edges**: `geom_segment(data = edge_df, aes(x, y, xend, yend), color = "grey80", linewidth = 0.3, alpha = 0.5)`
4. **Nodes (non-singleton)**: `geom_point(data = node_df[!node_df$is_singleton, ], aes(x, y, shape = fe_type, color = fe_type), size = 3)`
5. **Nodes (singleton)**: `geom_point(data = node_df[node_df$is_singleton, ], aes(x, y, shape = fe_type), color = "red", size = 4, stroke = 1.5)` -- singletons highlighted in red with larger size
6. **Shape scale**: `scale_shape_manual(values = ...)` -- use distinct shapes per FE type (circle = 16 for units, square = 15 for time, triangle = 17 for 3rd FE, diamond = 18 for 4th, etc.)
7. **Color scale**: If `color` parameter is provided by user, use it. Otherwise default: units = "#E41A1C" (red), time = "#377EB8" (blue), 3rd FE = "#4DAF4A" (green), 4th = "#984EA3" (purple). Use `scale_color_manual()`.
8. **Title**: `ggtitle(main)` where `main` defaults to `"Network Structure"` if NULL
9. **Axis labels**: Remove axis labels and ticks (`theme(axis.text = element_blank(), axis.ticks = element_blank())`) since coordinates are abstract layout positions
10. **Legend**: Show if `legendOff` is FALSE. Legend maps shapes and colors to FE dimension names.
11. **Font sizes**: Apply `cex.main`, `cex.legend` from `s`.

#### 7e. Print the plot

```r
print(p)
```

### Step 8: Construct return value

```r
result <- list(
    graph = g,
    singletons = singletons_df,
    components = comp_info
)
invisible(result)
```

---

## 4. Input Validation

### 4.1 In `.pv_plot_network()`

1. **igraph availability**: checked in Step 1 (hard stop).
2. **`I` matrix existence**: If `I` is NULL after extraction from `s`, construct `I <- matrix(1, TT, N)` (balanced panel fallback).
3. **`fe` parameter validation**: If `fe` is not NULL, check:
   - `fe` is a character vector
   - All elements of `fe` are column names in `data`
   - No element of `fe` duplicates `index[1]` or `index[2]`
   - Error message: `stop("'fe' must be a character vector of column names in the data, distinct from 'index'.", call. = FALSE)`
4. **Degenerate panel**: If N < 1 or TT < 1, stop with informative error.
5. **No observations**: If `sum(I) == 0`, stop: no observations in the panel.

### 4.2 In `panelview()` (modifications)

1. When `type == "network"`, skip formula/Y/D validation. Set `ignore.treat <- TRUE`.
2. The `fe` parameter must be passed through `s` (it is captured by `as.list(environment())`).

---

## 5. Output Contract

`.pv_plot_network(s)` returns (invisibly) a list with exactly three named elements:

| Name | Type | Description |
|------|------|-------------|
| `graph` | igraph object | The bipartite/k-partite graph with vertex attributes: `name`, `fe_type`, `component`, and edge attribute `weight` |
| `singletons` | data.frame | Columns: `node` (character), `fe_type` (character), `degree` (integer = 1). Zero rows if no singletons. |
| `components` | data.frame | Columns: `component` (integer), `size` (integer). One row per connected component. |

The function also prints (side effect) a ggplot2 plot to the active graphics device.

---

## 6. Numerical Constraints

- The observation matrix `I` is binary (0/1). No floating-point issues.
- Edge weights in k-partite case are positive integers (counts). No precision issues.
- Layout coordinates from igraph are floating-point but only used for plotting -- no numerical tolerance concerns.
- `grDevices::chull()` is robust for degenerate inputs (returns input indices for <= 2 points, but we only call it for 3+ points).

---

## 7. Implementation Notes (R-package profile)

### 7.1 File placement
- New file: `R/plot-network.R`
- Modified files: `R/panelView.R`, `DESCRIPTION`

### 7.2 DESCRIPTION changes
- Add `igraph` to `Suggests` field (not Imports): `Suggests: testthat (>= 3.0.0), igraph`

### 7.3 NAMESPACE
- No changes needed. igraph is in Suggests, accessed via `igraph::` namespace qualification. `.pv_plot_network` is internal (not exported -- starts with `.`).

### 7.4 igraph namespace qualification
ALL igraph function calls MUST be `igraph::function_name()`. Never use `@import igraph` or `importFrom`. This is required because igraph is in Suggests (conditional dependency).

### 7.5 Coding style
- Follow existing panelView conventions: `.pv_plot_*` naming, `with(s, {...})` pattern for accessing environment list, `ggplot2` for rendering.
- Use `grDevices::chull()` -- grDevices is a base package, no import needed.

### 7.6 Roxygen
- Update `man/panelview.Rd` to document `type = "network"` and the `fe` parameter. Since the package uses hand-written .Rd files (no roxygen2 detected in DESCRIPTION), the builder should edit `man/panelview.Rd` directly.

### 7.7 The `.pv_plot_network` function should NOT use `with(s, {...})`
Looking at `plot-treat.R`, the entire function body is wrapped in `with(s, {...})`. However, for `plot-network.R`, since we need to call `igraph::` namespaced functions and build complex data structures, it is cleaner to extract needed variables at the top: `I <- s$I; N <- s$N; TT <- s$TT; ...` etc. Both patterns are acceptable -- the key is that `s` provides all inputs.

---

## 8. Edge Cases and Special Handling

| Case | Handling |
|------|----------|
| Balanced panel (all I[j,i] = 1) | Complete bipartite graph K_{N,TT}. No singletons. Single component. Hull covers all nodes. |
| Single unit (N = 1) | Graph has 1 unit node + TT time nodes. Unit has degree TT. Each time node has degree 1 (all singletons). 1 component. |
| Single time period (TT = 1) | Graph has N unit nodes + 1 time node. Time has degree N. Each unit has degree 1 (all singletons). 1 component. |
| Completely disconnected (each unit in unique period, no overlap) | N + TT nodes, N edges. N components of size 2. All nodes are singletons (degree 1). |
| Very large panel (N > 500 or TT > 100) | Plot may be crowded. No special handling needed -- ggplot2 and igraph layout handle it. Edge alpha is low (0.5) to reduce clutter. |
| k-partite with k = 2 (fe = NULL) | Standard bipartite. Use bipartite layout. |
| k-partite with k = 3+ | General graph. Use FR layout. |
| Component with 1 node | Isolated node (degree 0). Possible only in k-partite case if a category has no observations. Highlighted as point, no hull. |
| Component with 2 nodes | No hull drawn (chull needs 3+ points). Nodes are connected by edge, visually distinct. |
| `I` is NULL in `s` | Balanced panel -- construct `I <- matrix(1, TT, N)`. |

---

## 9. Summary of Changes

| File | Change |
|------|--------|
| `R/panelView.R` line 15 | Add `fe = NULL` parameter to function signature |
| `R/panelView.R` line 109 | Add `"network"` to `match.arg()` vector |
| `R/panelView.R` ~line 113 | Add guard: `if (type == "network") ignore.treat <- TRUE` |
| `R/panelView.R` lines 1150-1161 | Add `type == "network"` branch to dispatch, before other types |
| `R/plot-network.R` (new) | Implement `.pv_plot_network(s)` per algorithm above |
| `DESCRIPTION` line 19 | Add `igraph` to Suggests |
| `man/panelview.Rd` | Document `type = "network"`, `fe` parameter, return value |
