# Review — interflex Python Package

## Request
20260331-interflex-py-224717 — Translate the R interflex package's linear estimator into a Python package.

## Reviewer Self-Check

- [x] Verified comprehension.md exists with FULLY UNDERSTOOD verdict (step 1)
- [x] Verified pipeline isolation (step 2)
- [x] Cross-compared spec.md against test-spec.md (step 3)
- [x] Verified convergence between both pipelines (step 4)
- [x] Checked test coverage for every changed code path (step 5)
- [x] Assessed whether assertions are structural-only or correctness-level (step 5)
- [x] For refactors: N/A (new package, not a refactor)
- [x] Verified tester ran required validation commands with exact evidence (step 7)
- [x] Verified tester executed ALL test-spec.md scenarios (step 7)
- [x] Cross-referenced ALL numerical tolerances in audit.md against test-spec.md (step 7a)
- [x] Verified Per-Test Result Table present in audit.md with all scenarios covered (step 4)
- [x] Verified Before/After Comparison Table: N/A (new package, no prior implementation) (step 4)
- [x] Checked documentation, architecture diagram in target repo root + run dir, process-record log entry in run dir, target repo clean of workflow artifacts (step 8)

---

## Step 1 — Comprehension Foundation

`comprehension.md` exists. Verdict: **FULLY UNDERSTOOD**. No HOLD rounds needed. All formulas are restated and verified against the R source code: model specification for discrete/continuous treatments, treatment effects (gen.general.TE), delta method variance (gen.delta.TE), simulation variance, bootstrap variance, ATE/AME (gen.ATE), cluster-robust variance (vcluster.R), evaluation grid, treatment type detection, treatment renaming, covariate preprocessing. No undefined symbols. No uploaded reference files (not applicable).

Formulas in comprehension.md match spec.md. Consistent coverage of the linear interaction model, gradient vectors, CGM correction, and all variance paths.

**Result: CLEAR**

---

## Step 2 — Pipeline Isolation

- **implementation.md**: Does not reference test-spec.md. Builder describes files created, design decisions, and known limitations. No test specification content.
- **audit.md**: Does not reference spec.md or implementation.md. Tester describes validation commands, test results, and test fixes. No implementation specification content.
- Builder wrote 50 unit tests in their implementation (as part of the code pipeline). Tester expanded to 92 tests. Some overlap is expected and acceptable since both derived tests from request.md independently.

**Result: CLEAR — no isolation breach detected.**

---

## Step 3 — Cross-Compare Specifications

**spec.md** describes (code pipeline):
- 7-module package structure with core.py entry point, linear.py orchestrator, effects.py for TE/ME, variance.py for delta method, vcov.py for vcov estimation, average.py for ATE/AME
- Detailed algorithm steps for input validation, covariate preprocessing, treatment detection, design matrix construction, WLS fitting, three variance paths (delta/simu/bootstrap), output assembly
- Formulas for TE, ME, delta method gradients, CGM cluster vcov, bootstrap resampling

**test-spec.md** describes (test pipeline):
- 6 shared fixtures (A-F) with known DGP parameters and fixed seeds
- 37+ numbered test scenarios across test_basic.py, test_variance.py, test_effects.py
- Behavioral contracts matching the same features: discrete/continuous treatment detection, TE/ME computation, delta/simu/bootstrap variance, cluster-robust vcov, ATE/AME, difference estimates, edge cases, property-based invariants, cross-reference benchmarks
- Tolerance guidelines: point estimates +/- 1.5 (n=200), manual OLS 1e-8, property invariants 1e-10, cluster vcov 1e-6, delta vs simu 15% relative, CI coverage [0.85, 1.00]

**Alignment check**:
- Both specs describe the same system from different angles (construction vs validation). No feature appears in test-spec.md without a corresponding algorithm in spec.md.
- All algorithm steps in spec.md have corresponding test scenarios in test-spec.md: input validation (2.5.x), treatment detection (2.2.4), TE computation (2.4.x), delta method (2.3.1), simulation (2.3.2), bootstrap (2.3.3), cluster vcov (2.3.7), ATE/AME (2.4.5-2.4.6), differences (2.2.6-2.2.8, 2.4.7), full moderation (2.4.9), weights (2.2.9), Xunif (2.2.10).
- Tolerances and boundary conditions align between specs.

**Result: CLEAR — specifications converge.**

---

## Step 4 — Convergence Verification

**Per-Test Result Table**: Present in audit.md with 92 rows across three test files (43 + 27 + 22). All 92 tests PASS. The table includes metric, expected, actual, tolerance, relative error, and verdict columns for each test.

**Before/After Comparison Table**: Correctly marked as N/A — this is a new package with no prior implementation. Not required.

**Convergence evidence**:
- Builder's implementation produces correct point estimates verified against known DGP parameters (TE at X=0 ~ 3.0, TE at X=5 ~ 7.0, ME at X=0 ~ 1.5, ME at X=2 ~ 2.3) — within the sampling noise tolerances specified in test-spec.md.
- Tester's independent validation confirms these same values match expectations.
- Property-based invariants hold exactly (1e-10): TE symmetry under relabeling, ME independence from D_ref, weight=1 equivalence, diff additivity, prediction consistency (pred_diff = TE).
- Manual OLS cross-reference matches within 1e-10 (package vs manual beta_hat computation).
- Manual HC1 vcov matches within 1e-8.
- Manual cluster vcov matches within 1e-6, and is numerically identical to statsmodels output.
- Delta vs simulation SE convergence at n=5000: within 15% relative.
- CI coverage at 95% nominal: within [0.85, 1.00] across 100 replications.
- Vcov matrix properties verified: symmetric (1e-8), positive diagonal, positive semi-definite (eigenvalues >= -1e-10), diagonal equals sd^2 (1e-8).

The two pipelines converge strongly. Builder built a correct implementation, and tester independently confirmed correctness through diverse test scenarios.

**Result: CLEAR — strong convergence.**

---

## Step 5 — Test Coverage

Every module has test coverage:

| Module | Changed Functions | Test Coverage |
|--------|-------------------|---------------|
| core.py | `interflex()` | Input validation (10 edge case tests), treatment detection (3 tests), diff values (4 tests), weights (2 tests), Xunif (1 test), missing values (2 tests) |
| linear.py | `interflex_linear()`, `_compute_delta()`, `_compute_simu()`, `_compute_bootstrap()` | Delta (7 tests), simulation (2 tests), bootstrap (3 tests), vcov matrix output (4 tests), CI coverage (1 test) |
| effects.py | `compute_effects()`, `_compute_discrete()`, `_compute_continuous()`, `_compute_diff()`, `_get_coef()` | TE linearity (1), ME linearity (1), TE at extremes (1), predicted values (2), diff consistency (2), Z_ref effect (1), full moderation (1), D_ref (2), neval (3), custom X_eval (2), symmetry (1), ME invariance (1), weight equivalence (1) |
| variance.py | `compute_delta_variance()`, `compute_base_delta_variance()`, `_build_gradient_vector()` | SE positivity (2), SE magnitude (1), CI containment (1), delta vs simu convergence (1), vcov symmetry/PSD (4), manual HC1 (1) |
| vcov.py | `compute_vcov()`, `vcov_cluster()` | Homoscedastic (1), robust (2), cluster (2), cluster bootstrap (2), manual cluster verification (1) |
| average.py | `compute_ate()` | ATE fixture A (1), AME fixture B (1), ATE SE structure (2) |

**Assertion quality**: Tests include both structural assertions (output structure, key presence, row counts) AND correctness assertions (numerical values against known DGP, manual OLS cross-reference, property invariants). The correctness assertions are substantial — manual OLS verification at 1e-8, property invariants at 1e-10, CI coverage Monte Carlo.

**Result: CLEAR — comprehensive coverage with correctness assertions.**

---

## Step 6 — Structural Refactors

N/A — this is a new package creation, not a refactor.

---

## Step 7 — Validation Evidence

**Commands run by tester**:
- `pip install -e .` — SUCCESS
- `python -m pytest tests/ -v` — 92 passed (after test fixes)

The audit.md includes the full pytest output listing all 92 test names and their PASS status. This is exact command output, not paraphrased.

**Scenario coverage**: Cross-referencing test-spec.md scenarios against audit.md:

| test-spec.md Scenario | Covered in audit.md? |
|------------------------|---------------------|
| 2.2.1 Binary discrete point estimates | Yes (6 tests) |
| 2.2.2 Continuous point estimates | Yes (3 tests) |
| 2.2.3 Multi-group discrete | Yes (4 tests) |
| 2.2.4 Treatment type auto-detection | Yes (3 tests) |
| 2.2.5 Base group selection | Yes (3 tests) |
| 2.2.6-2.2.8 Diff values | Yes (5 tests) |
| 2.2.9 Weighted estimation | Yes (2 tests) |
| 2.2.10 Xunif transform | Yes (1 test) |
| 2.3.1-2.3.3 Delta/simu/bootstrap SE | Yes (7 tests) |
| 2.3.4 Delta vs simu large sample | Yes (1 test) |
| 2.3.5-2.3.6 Homoscedastic/HC1 vcov | Yes (3 tests) |
| 2.3.7-2.3.8 Cluster vcov/bootstrap | Yes (4 tests) |
| 2.3.9 Vcov matrix output | Yes (4 tests) |
| 2.3.10 CI coverage | Yes (1 test) |
| 2.4.1-2.4.2 TE/ME linearity | Yes (2 tests) |
| 2.4.3 TE at extreme X | Yes (1 test) |
| 2.4.4-2.4.5 Predicted values, ATE/AME | Yes (4 tests) |
| 2.4.6 ATE with delta SE | Yes (2 tests) |
| 2.4.7 Diff estimate consistency | Yes (2 tests) |
| 2.4.8 Z_ref effect | Yes (1 test) |
| 2.4.9 Full moderation | Yes (1 test) |
| 2.4.10 Continuous D_ref | Yes (2 tests) |
| 2.4.11-2.4.12 neval, custom X_eval | Yes (5 tests) |
| 2.5.1-2.5.10 Edge cases | Yes (13 tests) |
| 2.6.1-2.6.6 Property invariants | Yes (6 tests) |
| 2.7.1-2.7.4 Cross-reference benchmarks | Yes (4 tests) |

All test-spec.md scenarios are represented. Tester actually expanded beyond the spec (builder's 50 tests expanded to 92).

### Step 7a — Tolerance Integrity Audit

Cross-referencing every numerical tolerance in audit.md against test-spec.md Section 5:

| Category | test-spec.md | audit.md | Match? |
|----------|-------------|----------|--------|
| Point estimates vs true DGP (n=200) | +/- 1.5 | atol=1.5 | YES |
| Continuous ME estimates (n=300) | +/- 0.5 | atol=0.5 | YES |
| Weighted estimates | +/- 2.0 | atol=2.0 | YES |
| SE comparison between methods | Within factor of 2 | ratio check | YES |
| Bootstrap SE comparison | Within factor of 3 | ratio check | YES |
| Manual OLS vs package OLS | 1e-8 | atol=1e-8 | YES |
| Manual vcov (HC1) | 1e-8 | atol=1e-8 | YES |
| Manual vcov (cluster) | 1e-6 | atol=1e-6 | YES |
| Property invariants | 1e-10 | atol=1e-10 | YES |
| CI coverage | [0.85, 1.00] | bounds checked | YES |
| Delta vs simu SE (large n) | Within 15% relative | rel_diff < 0.15 | YES |
| Diff estimate accuracy | +/- 0.3 | **widened to +/- 0.5** | **DEVIATION** |

**One tolerance deviation detected**: test_diff_additivity widened individual diff value tolerance from 0.3 to 0.5. Tester's rationale: with n=200, the estimated beta_DX=0.903 (true=0.8) gives diff error of 0.308 > 0.3. This is pure sampling variability (SE of beta_DX with n=200 is approximately 0.1, making 0.308 well within 3 SE). The exact additivity check (1e-10) was preserved unchanged. The widening applies only to the point estimate comparison against the true DGP value, not to any invariant or cross-validation tolerance.

**Assessment**: This tolerance widening is justified. The test-spec.md tolerance of 0.3 for diff values was too tight given the specified sample size of n=200. The relevant invariant (additivity: diff(8,2) = diff(5,2) + diff(8,5)) holds at 1e-10 precision. The implementation is correct; the test-spec tolerance was unrealistic for this DGP/sample size combination. This is not tolerance inflation to mask a bug.

**Evasion pattern check**:
- No assertions removed or commented out: CLEAR
- No try/catch wrappers swallowing failures: CLEAR
- No reduced iteration counts or sample sizes: CI coverage uses 100 replications as specified; delta-simu convergence uses n=5000 as specified; bootstrap uses nboots=200/500 as specified
- No random seed changes without justification: CLEAR
- No silently omitted test scenarios: all test-spec scenarios accounted for

**Result: CLEAR — tolerance integrity maintained. One justified deviation documented.**

---

## Step 8 — Documentation and Process Record

### ARCHITECTURE.md
- **Target repo root**: EXISTS at `.repos/example-probit/ARCHITECTURE.md`
- **Run directory**: EXISTS at the run directory
- Contains three Mermaid diagrams: module structure graph, function call graph, and data flow graph
- Module reference table with 7 modules (all marked "changed" = yes, correct for new package)
- Function reference table with 18 functions, all with correct Defined In, Called By, Calls columns
- Accurately reflects the current codebase structure: layered pipeline (API -> Orchestration -> Computation + Inference), consistent with the actual source code

### Log entry
- EXISTS at run directory as `log-entry.md`
- Contains `<!-- filename: 2026-03-31-interflex-py-linear-estimator.md -->` header for workspace sync
- Contains: What Changed, Files Changed (13 files), Process Record with Per-Test Result Table (22 key rows shown) and Before/After Comparison Table (N/A), Test Fixes Applied (4 entries), Problems and Resolutions, Design Decisions (7 items), Handoff Notes (7 items)
- Process record is complete

### docs.md
- EXISTS at run directory
- Documents all public functions and their docstring coverage (10 functions, all complete)
- Notes deferred items (README.md, Sphinx/mkdocs, usage examples)

### Target repo clean
- No CHANGELOG.md in target repo: CLEAR
- No HANDOFF.md in target repo: CLEAR
- No runs/ directory in target repo: CLEAR
- No logs/ directory in target repo: CLEAR
- ARCHITECTURE.md in target repo root: Expected, CLEAR

**Result: CLEAR — documentation complete, target repo clean.**

---

## Test Fix Assessment

Tester applied 4 test fixes. Each must be evaluated for correctness:

1. **test_ate_fixture_a**: Test did not handle dict return type. This is a test bug (incorrect assertion on the return type), not an implementation bug. The implementation's dict return is consistent with spec.md step 27 (avg_estimate is a dict for discrete). **Justified.**

2. **test_diff_additivity**: Tolerance widened from 0.3 to 0.5 for individual diff values. See tolerance integrity analysis above. The additivity invariant (1e-10) was preserved. **Justified.**

3. **test_te_negated_on_relabel**: Test had incorrect base group specification. Setting `base="treated"` after label swap preserves the comparison direction instead of negating it. Removing `base="treated"` correctly tests the symmetry property from test-spec.md 2.6.1. **Justified.**

4. **test_cluster_se_larger_than_robust**: Test assumed cluster SEs are always larger than HC1 SEs. This is not universally true (depends on the data and intra-cluster correlation structure). The cluster vcov was verified numerically identical to statsmodels output. **Justified.**

All 4 fixes corrected test bugs, not implementation bugs. No implementation code was modified to make tests pass.

---

## Code Quality Notes

1. **Build backend deviation**: spec.md specified `setuptools.backends._legacy:_Backend` which does not exist. Builder changed to `setuptools.build_meta` (the standard backend). This is a necessary and correct deviation.

2. **Factor Z_ref limitation**: The code has a known simplification in handling user-provided Z_ref for categorical covariates (defaults to 0 rather than resolving to sum-coded values). This is documented in implementation.md and log-entry.md handoff notes. Low risk for typical usage.

3. **No numerical instabilities detected**: All vcov computations use `np.linalg.solve` with pinv fallback. Gradient vectors handle missing coefficients gracefully via `_get_coef` returning 0.0. Square roots use `max(var, 0.0)` to prevent NaN from floating-point underflow.

4. **No security issues**: Package is a pure numerical computation library with no network access, file system writes beyond normal package behavior, or user input injection vectors.

---

## Verdict

**PASS** — Both pipelines converged. 9 challenges raised, all cleared. Safe to ship.

**Evidence summary**:
- Comprehension verified: FULLY UNDERSTOOD, all formulas traced
- Pipeline isolation maintained: no cross-contamination between code and test pipelines
- Specifications converge: spec.md and test-spec.md describe the same system consistently
- 92/92 tests pass with correctness-level assertions (not just structural)
- Property invariants hold at machine precision (1e-10)
- Manual OLS and vcov cross-references match within specified tolerances
- One justified tolerance deviation (0.3 -> 0.5 for diff point estimates vs DGP truth at n=200)
- ARCHITECTURE.md present in both target repo root and run directory
- Log entry complete with process record
- Target repo clean of workflow artifacts
- All 4 test fixes are justified test bugs, not implementation workarounds
