# Test Specification: `type = "network"`

## Request ID
REQ-20260401-network-viz

## Profile
r-package

---

## 1. Behavioral Contract

The `panelview()` function, when called with `type = "network"`, MUST:

1. **Produce a bipartite network plot** showing units and time periods as distinct node types
2. **Highlight singletons** (degree-1 nodes) visually in the plot
3. **Draw convex hulls** around connected components (for components with 3+ nodes)
4. **Support k-partite graphs** when additional FE column names are provided via the `fe` parameter
5. **Require igraph** from Suggests -- fail gracefully with informative error if not installed
6. **Return invisibly** a list with three named elements: `graph` (igraph object), `singletons` (data.frame), `components` (data.frame)
7. **Not require** a formula, outcome, or treatment variable (network type only needs panel structure)
8. **Work with both balanced and unbalanced panels**

---

## 2. Test Scenarios

### 2.1 Basic bipartite graph from balanced panel

**Input**: A small balanced panel (4 units x 3 periods):

```r
set.seed(42)
df_balanced <- data.frame(
    unit = rep(1:4, each = 3),
    time = rep(1:3, 4),
    y = rnorm(12)
)
```

**Expected behavior**:
- Function runs without error
- Returns a list with names `graph`, `singletons`, `components`
- `graph` is an igraph object with 7 vertices (4 units + 3 time periods) and 12 edges (complete bipartite K_{4,3})
- `singletons` is a data.frame with 0 rows (balanced panel has no singletons -- every unit appears in 3 periods, every period has 4 units)
- `components` is a data.frame with 1 row (single connected component of size 7)
- All vertices have degree >= 2

### 2.2 Unbalanced panel with singletons

**Input**: Construct a panel where some units appear in only one period:

```r
df_unbal <- data.frame(
    unit = c(1, 1, 1, 2, 2, 3, 4, 4, 5),
    time = c(1, 2, 3, 1, 2, 3, 4, 5, 6),
    y = rnorm(9)
)
```

Structure:
- Unit 1: periods 1,2,3 (degree 3)
- Unit 2: periods 1,2 (degree 2)
- Unit 3: period 3 only (degree 1 -- SINGLETON)
- Unit 4: periods 4,5 (degree 2)
- Unit 5: period 6 only (degree 1 -- SINGLETON)
- Period 1: units 1,2 (degree 2)
- Period 2: units 1,2 (degree 2)
- Period 3: units 1,3 (degree 2)
- Period 4: unit 4 only (degree 1 -- SINGLETON)
- Period 5: unit 4 only (degree 1 -- SINGLETON)
- Period 6: unit 5 only (degree 1 -- SINGLETON)

**Expected behavior**:
- Returns without error
- `singletons` data.frame has 5 rows: unit 3, unit 5, period 4, period 5, period 6
- `singletons$degree` is all 1
- Graph has 11 vertices (5 units + 6 periods), 9 edges
- `components` has 3 components:
  - Component containing units 1,2,3 and periods 1,2,3 (size 6)
  - Component containing unit 4 and periods 4,5 (size 3)
  - Component containing unit 5 and period 6 (size 2)

### 2.3 No formula required

**Input**:

```r
df <- data.frame(unit = rep(1:3, each = 2), time = rep(1:2, 3), y = 1:6)
result <- panelview(data = df, index = c("unit", "time"), type = "network")
```

**Expected behavior**: No error. Function should NOT require a formula argument for network type.

Also test that it works when formula IS provided (should be ignored gracefully):

```r
result2 <- panelview(y ~ 1, data = df, index = c("unit", "time"), type = "network")
```

### 2.4 Return value structure

**Input**: Any valid panel data call with `type = "network"`.

**Expected behavior**:
- Return value is a list (verify with `is.list()`)
- Has exactly names: `"graph"`, `"singletons"`, `"components"`
- `result$graph`: `igraph::is.igraph(result$graph)` is TRUE
- `result$singletons`: `is.data.frame(result$singletons)` is TRUE, has columns `node`, `fe_type`, `degree`
- `result$components`: `is.data.frame(result$components)` is TRUE, has columns `component`, `size`

### 2.5 Large unbalanced panel (using package data)

**Input**: Use the `turnout` dataset bundled with panelView:

```r
data(panelView)
result <- panelview(data = turnout, index = c("abb", "year"), type = "network")
```

**Expected behavior**:
- Runs without error
- Returns list with correct structure
- Graph vertex count = number of unique states + number of unique years
- `components` has at least 1 row
- Plot is produced (no graphics device errors)

### 2.6 Vertex attributes are correct

**Input**: The balanced panel from 2.1.

**Expected behavior**:
- `igraph::V(result$graph)$name` has length 7
- `igraph::V(result$graph)$fe_type` contains exactly two distinct values matching the index column names ("unit" and "time")
- `igraph::V(result$graph)$component` is all the same value (1 component)
- Unit nodes: `fe_type` == index[1]
- Time nodes: `fe_type` == index[2]

---

## 3. Edge Case Scenarios

### 3.1 Single unit

**Input**:

```r
df_one_unit <- data.frame(unit = rep(1, 3), time = 1:3, y = rnorm(3))
result <- panelview(data = df_one_unit, index = c("unit", "time"), type = "network")
```

**Expected behavior**:
- No error
- Graph has 4 vertices (1 unit + 3 time periods)
- Unit node has degree 3 (NOT a singleton)
- Each time node has degree 1 (ALL are singletons)
- `singletons` has 3 rows (all time periods)
- 1 connected component

### 3.2 Single time period

**Input**:

```r
df_one_time <- data.frame(unit = 1:5, time = rep(1, 5), y = rnorm(5))
result <- panelview(data = df_one_time, index = c("unit", "time"), type = "network")
```

**Expected behavior**:
- No error
- Graph has 6 vertices (5 units + 1 time)
- Time node has degree 5 (NOT a singleton)
- Each unit node has degree 1 (ALL are singletons)
- `singletons` has 5 rows
- 1 connected component

### 3.3 Fully disconnected panel (each unit-period pair is unique)

**Input**:

```r
df_disconn <- data.frame(unit = 1:4, time = 1:4, y = rnorm(4))
```

(Each unit observed in exactly one unique period, no period shared.)

**Expected behavior**:
- Graph has 8 vertices, 4 edges
- ALL nodes are singletons (degree 1): `nrow(result$singletons) == 8`
- 4 connected components, each of size 2
- `nrow(result$components) == 4`

### 3.4 igraph not available

**Input**: Mock `requireNamespace("igraph", quietly = TRUE)` to return FALSE (use `testthat::local_mocked_bindings` or `mockr` or simply test the error message pattern).

**Expected behavior**: Error message contains "igraph" and "install.packages".

Practical approach: Use `expect_error()` with a regex pattern. If igraph IS installed in the test environment (which it likely is), this test can be skipped with `testthat::skip()` or tested by temporarily unloading the namespace -- but the simplest approach is to verify the check is present by inspecting the source. Alternatively, test by calling `.pv_plot_network` directly with a mock.

### 3.5 Invalid `fe` parameter

**Input**:

```r
df <- data.frame(unit = rep(1:3, each = 2), time = rep(1:2, 3), y = 1:6)
```

Test cases:
- `fe = 123` (not character): expect error
- `fe = "nonexistent"` (column not in data): expect error
- `fe = "unit"` (duplicates index[1]): expect error

### 3.6 Two units, two time periods (minimal complete bipartite)

**Input**:

```r
df_2x2 <- data.frame(unit = c(1,1,2,2), time = c(1,2,1,2), y = 1:4)
result <- panelview(data = df_2x2, index = c("unit", "time"), type = "network")
```

**Expected behavior**:
- Graph: 4 vertices, 4 edges (K_{2,2})
- 0 singletons
- 1 component of size 4

---

## 4. k-partite Extension Scenarios

### 4.1 Three-way FE (unit x time x region)

**Input**:

```r
df_3way <- data.frame(
    unit = c(1, 1, 2, 2, 3, 3),
    time = c(1, 2, 1, 2, 3, 4),
    region = c("A", "A", "A", "B", "B", "B"),
    y = rnorm(6)
)
result <- panelview(data = df_3way, formula = y ~ 1,
                    index = c("unit", "time"), fe = "region",
                    type = "network")
```

**Expected behavior**:
- No error
- `igraph::V(result$graph)$fe_type` contains 3 distinct values: "unit", "time", and "region"
- Graph has nodes for unique units (3) + unique times (4) + unique regions (2) = 9 nodes
- Each observation creates edges between its unit-time, unit-region, and time-region pairs
- Return structure is the same: `graph`, `singletons`, `components`

### 4.2 k-partite with fe = character vector of length 2

**Input**:

```r
df_4way <- data.frame(
    unit = c(1, 1, 2, 2),
    time = c(1, 2, 1, 2),
    region = c("A", "A", "B", "B"),
    sector = c("X", "Y", "X", "Y"),
    y = rnorm(4)
)
result <- panelview(data = df_4way, formula = y ~ 1,
                    index = c("unit", "time"), fe = c("region", "sector"),
                    type = "network")
```

**Expected behavior**:
- `igraph::V(result$graph)$fe_type` contains 4 distinct values
- No error
- Correct return structure

---

## 5. Property-Based Invariants

### 5.1 Vertex count invariant

For any bipartite (k=2) call: `igraph::vcount(result$graph) == N_unique_units + N_unique_periods`

### 5.2 Edge count invariant (bipartite)

For bipartite: `igraph::ecount(result$graph) == sum(I)` where I is the observation matrix (number of non-missing unit-period pairs).

### 5.3 Bipartite property (k=2)

For bipartite calls: `igraph::is_bipartite(result$graph)` must be TRUE.

### 5.4 Singleton degree invariant

All nodes in `result$singletons` must have `igraph::degree(result$graph, v) == 1`.

### 5.5 Component membership consistency

Every vertex must belong to exactly one component: `length(unique(igraph::V(result$graph)$component)) == nrow(result$components)`.

### 5.6 Component sizes sum to vertex count

`sum(result$components$size) == igraph::vcount(result$graph)`

### 5.7 Balanced panel: no singletons

If the input panel is balanced (N units x TT periods, all observed), then `nrow(result$singletons) == 0`.

### 5.8 Balanced panel: one component

If the input panel is balanced, `nrow(result$components) == 1`.

---

## 6. Regression Scenarios

Not applicable -- this is a new feature, not a bug fix.

---

## 7. Cross-Reference Benchmarks

### 7.1 Manual graph verification

For the unbalanced panel in Scenario 2.2, the graph structure can be verified by hand:
- Adjacency: unit 1 connects to periods 1,2,3; unit 2 to periods 1,2; unit 3 to period 3; unit 4 to periods 4,5; unit 5 to period 6
- Components: {u1, u2, u3, t1, t2, t3}, {u4, t4, t5}, {u5, t6}
- Singletons: u3, u5, t4, t5, t6

### 7.2 igraph independent verification

For any test case, independently construct the graph using `igraph::graph_from_biadjacency_matrix()` on the known I matrix, compute `igraph::degree()` and `igraph::components()`, and compare against the function's return values.

---

## 8. Validation Commands

```r
# Run unit tests
testthat::test_dir("tests/testthat")

# R CMD check (full package check)
R CMD build . && R CMD check *.tar.gz --as-cran

# Specific test file
testthat::test_file("tests/testthat/test-network.R")
```

### R CMD check expectations
- 0 errors
- 0 warnings
- 0 notes (except possible NOTE about igraph in Suggests not being used in examples -- acceptable)

---

## 9. Test File Location

All tests for this feature should be in: `tests/testthat/test-network.R`

### Test helper pattern

Follow the existing pattern in `test-panelview.R`:

```r
## Helper: run panelview() suppressing the plot device output
pv_net <- function(...) {
    pdf(NULL)
    on.exit(dev.off())
    panelview(..., type = "network")
}
```

This suppresses graphics device output during testing. The helper wraps `panelview()` with `type = "network"` and captures the invisible return value.

---

## 10. Test Organization

| Test Group | Scenarios |
|-----------|-----------|
| Basic functionality | 2.1, 2.3, 2.4, 2.5 |
| Singleton detection | 2.2, 3.1, 3.2, 3.3 |
| Connected components | 2.2, 3.3, 3.6 |
| k-partite | 4.1, 4.2 |
| Edge cases | 3.1, 3.2, 3.3, 3.4, 3.5, 3.6 |
| Return value structure | 2.4, 2.6 |
| Invariants | 5.1 through 5.8 |
