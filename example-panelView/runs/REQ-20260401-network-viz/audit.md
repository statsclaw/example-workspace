# Audit Report

## Request ID
REQ-20260401-network-viz

## Verdict: PASS

All 78 network tests pass (0 failures), all 36 existing panelview tests pass (0 regressions), and R CMD check --as-cran produces 0 ERRORs.

---

## 1. Per-Test Result Table

| Test | Metric | Expected | Actual | Tolerance | Rel. Error | Verdict |
|------|--------|----------|--------|-----------|------------|---------|
| 2.1 balanced panel | vertex count | 7 | 7 | exact | -- | PASS |
| 2.1 balanced panel | edge count | 12 | 12 | exact | -- | PASS |
| 2.1 balanced panel | singleton count | 0 | 0 | exact | -- | PASS |
| 2.1 balanced panel | component count | 1 | 1 | exact | -- | PASS |
| 2.1 balanced panel | component size | 7 | 7 | exact | -- | PASS |
| 2.1 balanced panel | min degree >= 2 | TRUE | TRUE | exact | -- | PASS |
| 2.1 balanced panel | is_igraph | TRUE | TRUE | exact | -- | PASS |
| 2.1 balanced panel | return names | graph,singletons,components | graph,singletons,components | exact | -- | PASS |
| 2.2 unbalanced panel | vertex count | 11 | 11 | exact | -- | PASS |
| 2.2 unbalanced panel | edge count | 9 | 9 | exact | -- | PASS |
| 2.2 unbalanced panel | singleton count | 5 | 5 | exact | -- | PASS |
| 2.2 unbalanced panel | all singleton degrees = 1 | TRUE | TRUE | exact | -- | PASS |
| 2.2 unbalanced panel | component count | 3 | 3 | exact | -- | PASS |
| 2.2 unbalanced panel | component sizes (sorted) | c(6,3,2) | c(6,3,2) | exact | -- | PASS |
| 2.3 no formula | no error without formula | no error | no error | exact | -- | PASS |
| 2.3 no formula | no error with formula | no error | no error | exact | -- | PASS |
| 2.3 no formula | returns list | TRUE | TRUE | exact | -- | PASS |
| 2.4 return structure | is.list | TRUE | TRUE | exact | -- | PASS |
| 2.4 return structure | names match | graph,singletons,components | graph,singletons,components | exact | -- | PASS |
| 2.4 return structure | is_igraph | TRUE | TRUE | exact | -- | PASS |
| 2.4 return structure | singletons is data.frame | TRUE | TRUE | exact | -- | PASS |
| 2.4 return structure | singletons columns | node,fe_type,degree | present | exact | -- | PASS |
| 2.4 return structure | components is data.frame | TRUE | TRUE | exact | -- | PASS |
| 2.4 return structure | components columns | component,size | present | exact | -- | PASS |
| 2.5 turnout data | no error | no error | no error | exact | -- | PASS |
| 2.5 turnout data | is_igraph | TRUE | TRUE | exact | -- | PASS |
| 2.5 turnout data | vertex count = states+years | n_states+n_years | n_states+n_years | exact | -- | PASS |
| 2.5 turnout data | components >= 1 | TRUE | TRUE | exact | -- | PASS |
| 2.6 vertex attributes | name count | 7 | 7 | exact | -- | PASS |
| 2.6 vertex attributes | fe_type distinct values | unit,time | unit,time | exact | -- | PASS |
| 2.6 vertex attributes | single component attr | 1 unique | 1 unique | exact | -- | PASS |
| 3.1 single unit | vertex count | 4 | 4 | exact | -- | PASS |
| 3.1 single unit | singleton count | 3 | 3 | exact | -- | PASS |
| 3.1 single unit | component count | 1 | 1 | exact | -- | PASS |
| 3.2 single time period | vertex count | 6 | 6 | exact | -- | PASS |
| 3.2 single time period | singleton count | 5 | 5 | exact | -- | PASS |
| 3.2 single time period | component count | 1 | 1 | exact | -- | PASS |
| 3.3 fully disconnected | vertex count | 8 | 8 | exact | -- | PASS |
| 3.3 fully disconnected | edge count | 4 | 4 | exact | -- | PASS |
| 3.3 fully disconnected | singleton count | 8 | 8 | exact | -- | PASS |
| 3.3 fully disconnected | component count | 4 | 4 | exact | -- | PASS |
| 3.3 fully disconnected | all components size 2 | TRUE | TRUE | exact | -- | PASS |
| **3.4 igraph check** | **error when igraph mocked unavailable** | **error mentioning "igraph"** | **error mentioning "igraph"** | **exact** | **--** | **PASS (fixed)** |
| 3.5 invalid fe (not char) | error raised | error | error | exact | -- | PASS |
| 3.5 invalid fe (missing col) | error raised | error | error | exact | -- | PASS |
| 3.5 invalid fe (dup index) | error raised | error | error | exact | -- | PASS |
| 3.6 minimal 2x2 | vertex count | 4 | 4 | exact | -- | PASS |
| 3.6 minimal 2x2 | edge count | 4 | 4 | exact | -- | PASS |
| 3.6 minimal 2x2 | singleton count | 0 | 0 | exact | -- | PASS |
| 3.6 minimal 2x2 | component count | 1 | 1 | exact | -- | PASS |
| 3.6 minimal 2x2 | component size | 4 | 4 | exact | -- | PASS |
| 4.1 three-way FE | is_igraph | TRUE | TRUE | exact | -- | PASS |
| 4.1 three-way FE | fe_type distinct count | 3 | 3 | exact | -- | PASS |
| 4.1 three-way FE | fe_types contain unit,time,region | TRUE | TRUE | exact | -- | PASS |
| 4.1 three-way FE | vertex count | 9 | 9 | exact | -- | PASS |
| 4.2 four-way FE | is_igraph | TRUE | TRUE | exact | -- | PASS |
| 4.2 four-way FE | fe_type distinct count | 4 | 4 | exact | -- | PASS |
| 4.2 four-way FE | return names | graph,singletons,components | graph,singletons,components | exact | -- | PASS |
| 5.1 vertex count invariant | holds for 3 datasets | TRUE | TRUE | exact | -- | PASS |
| 5.2 edge count invariant | holds for 3 datasets | TRUE | TRUE | exact | -- | PASS |
| 5.3 bipartite property | is_bipartite | TRUE | TRUE | exact | -- | PASS |
| 5.4 singleton degree | all singletons degree=1 | TRUE | TRUE | exact | -- | PASS |
| 5.4 singleton degree | count matches graph | TRUE | TRUE | exact | -- | PASS |
| 5.5 component membership | unique comps = nrow(components) | TRUE | TRUE | exact | -- | PASS |
| 5.6 component sizes sum | sum = vcount | TRUE | TRUE | exact | -- | PASS |
| 5.7 balanced no singletons | 0 singletons | 0 | 0 | exact | -- | PASS |
| 5.8 balanced one component | 1 component | 1 | 1 | exact | -- | PASS |

---

## 2. Validation Commands and Results

### 2.1 testthat::test_file("tests/testthat/test-network.R") -- via devtools::load_all

**Command**: `Rscript -e 'devtools::load_all("."); library(testthat); test_file("tests/testthat/test-network.R")'`

**Result**: FAIL 0 | WARN 2 | SKIP 0 | PASS 78

All 78 test assertions pass. The 2 warnings are cosmetic ggplot2 scale warnings on fully disconnected panels (test 3.3 and invariant 5.2) where all nodes are singletons, so `scale_color_manual` has no non-singleton data to match.

### 2.2 testthat::test_file("tests/testthat/test-panelview.R") -- regression check

**Command**: `Rscript -e 'devtools::load_all("."); library(testthat); test_file("tests/testthat/test-panelview.R")'`

**Result**: FAIL 0 | WARN 0 | SKIP 0 | PASS 36

All 36 existing tests pass. **No regressions**.

### 2.3 R CMD check --as-cran --no-manual

**Command**: `R CMD build . && R CMD check panelView_1.2.1.tar.gz --as-cran --no-manual`

**Result**: Status: 0 ERRORS, 1 WARNING, 1 NOTE

- **WARNING**: "Insufficient package version (submitted: 1.2.1, existing: 1.2.1)" -- expected CRAN version gate; would resolve when version is bumped for release.
- **NOTE**: Hidden `.git` directory found in tarball -- expected when building from a git worktree; would not appear in a release tarball built from a clean export.

All R CMD check items pass:
- Package can be installed: OK
- Dependencies in R code: OK
- R code for possible problems: OK (no unqualified igraph calls, no missing imports)
- S3 generic/method consistency: OK
- Rd files: OK
- Examples: OK
- Missing documentation entries: OK
- Tests: OK (all tests pass in installed package context)

### 2.4 DESCRIPTION verification

- igraph is listed in `Suggests: testthat (>= 3.0.0), igraph` -- **PASS**
- igraph is NOT in Imports -- **PASS**
- rlang is listed in `Imports: ggplot2 (>= 3.4.0), gridExtra, grid, dplyr (>= 1.0.0), rlang` -- **PASS**

### 2.5 NAMESPACE verification

- `importFrom("rlang", ".data")` is present -- **PASS**
- No igraph entries in NAMESPACE -- **PASS** (correctly uses Suggests + namespace-qualified calls)

### 2.6 igraph namespace qualification

- All igraph function calls in `R/plot-network.R` use `igraph::` prefix (20+ calls) -- **PASS**
- No `library(igraph)` or `require(igraph)` calls -- **PASS**
- `requireNamespace("igraph", quietly = TRUE)` used for graceful availability check -- **PASS**

---

## 3. Test 3.4 Fix Details

### Previous defect

Test 3.4 "graceful error when igraph not available" used `system.file("R", "plot-network.R", package = "panelView")` to read source code and verify the `requireNamespace("igraph")` check exists. This fails for installed packages because R compiles `.R` files into binary `.rdb`/`.rdx` archives -- the individual source files do not exist at that path.

### Fix applied

Replaced the source-file-reading approach with a behavioral test using `testthat::local_mocked_bindings()`:

```r
test_that("graceful error when igraph not available", {
    local_mocked_bindings(
        requireNamespace = function(package, ...) {
            if (package == "igraph") return(FALSE)
            base::requireNamespace(package, ...)
        },
        .package = "base"
    )
    df <- data.frame(unit = rep(1:3, each = 2), time = rep(1:2, 3), y = 1:6)
    expect_error(
        pv_net(data = df, index = c("unit", "time")),
        "igraph"
    )
})
```

This mocks `requireNamespace` to return `FALSE` specifically for igraph, then verifies that `.pv_plot_network()` produces an error message mentioning "igraph". This tests the actual behavior rather than inspecting source code, and works in both `devtools::load_all()` and `R CMD check --as-cran` contexts.

### Verification

- `devtools::load_all()` + `test_file()`: PASS (78/78)
- `R CMD check --as-cran`: tests OK (0 errors from test 3.4)

---

## 4. Original BLOCK Issues -- Fix Verification (unchanged from previous audit)

### 4.1 Single time period crash (FIXED)

**Original**: `panelview(data, index, type="network")` crashed with `"missing value where TRUE/FALSE needed"` when the panel had only 1 unique time period.

**Fix applied**: Builder added a guard `if (type != "network") { ... }` around the time gap validation block.

**Verification**: Test 3.2 passes. Returns 6 vertices (5 units + 1 time), 5 singletons, 1 component.

### 4.2 `.data` not imported (FIXED)

**Original**: R CMD check NOTE: `.pv_plot_network: no visible binding for global variable '.data'`.

**Fix applied**: Builder added `rlang` to Imports in DESCRIPTION and `importFrom("rlang", ".data")` to NAMESPACE.

**Verification**: R CMD check `* checking R code for possible problems ... OK`.

---

## 5. Warnings Observed

1. **ggplot2 color scale warning** on fully disconnected panels: cosmetic only -- occurs when ALL nodes are singletons so `scale_color_manual` has no matching data. Does not affect output correctness.

---

## 6. Before/After Comparison

| Issue | Previous Audit | Current Audit | Status |
|-------|---------------|---------------|--------|
| Single time period crash | ERROR | PASS | FIXED |
| `.data` not imported | NOTE | OK | FIXED |
| Test 3.4 source check | FAIL during R CMD check (test defect) | PASS (rewritten with mock) | FIXED |
| Existing tests regression | 36/36 PASS | 36/36 PASS | No regression |
| R CMD check | 1 ERROR, 1 NOTE | 0 ERRORS, 1 WARNING (version), 1 NOTE (.git) | CLEAN |

---

## 7. Additional Observations

### 7.1 Non-unique vertex names in bipartite case

In the bipartite (k=2) case, vertex names are not prefixed with the FE type (e.g., unit "3" and time period "3" both have `V(g)$name == "3"`). The k-partite case (k>2) correctly uses prefixed names. This inconsistency is noted for the reviewer but is not a functional issue -- the `fe_type` vertex attribute distinguishes them.

### 7.2 Warning count during R CMD check

R CMD check reports warnings during test execution. Most are from existing tests (ggplot2 deprecation warnings, treatment level messages, etc.) and are not related to the network feature. The 2 network-specific warnings (ggplot2 color scale on disconnected panels) are cosmetic.

---

## 8. Environment

- R version: 4.5.1 (2025-06-13)
- Platform: aarch64-apple-darwin20 (macOS Tahoe 26.5)
- igraph version: installed (namespace-qualified calls only)
- ggplot2 version: loaded via panelView
- testthat version: >= 3.0.0
- rlang version: loaded via panelView
- Test execution: worktree at `.claude/worktrees/agent-a8322fc8/` (builder's bug-fix worktree)

---

## 9. Signal

**No signal** -- all tests pass, R CMD check clean. Verdict is PASS.
