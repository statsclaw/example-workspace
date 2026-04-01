# Impact Analysis

## Request Summary
Create a new Python package `interflex` translating the linear estimator from R's interflex package.

## Source Analysis (R package xuyiqing/interflex)

### Files to Translate
| R File | Lines | Python Target | Complexity |
|--------|-------|---------------|------------|
| `R/interflex.R` (lines 1-950) | ~950 | `interflex/core.py` — input validation, treatment preprocessing | Medium |
| `R/linear.R` (full file) | 2249 | `interflex/linear.py` — core estimator | High |
| `R/vcluster.R` | 39 | `interflex/vcov.py` — cluster-robust covariance | Low |

### Key Algorithms to Translate
1. **Formula construction** — discrete treatment indicators + interactions, continuous D×X
2. **Model fitting** — OLS via statsmodels (equivalent to R's `glm(family=gaussian)`)
3. **Variance-covariance** — homoscedastic (`vcov`), HC1 robust (`vcovHC`), cluster-robust (`vcovCluster`)
4. **Treatment effects** — `gen.general.TE()`: point estimates at evaluation grid
5. **Delta method** — `gen.delta.TE()`: analytical gradient-based SEs via ∇f^T Σ ∇f
6. **Simulation variance** — Draw from N(β̂, Σ̂), compute TE/ME for each draw
7. **Bootstrap variance** — Resample data, refit, compute TE/ME per bootstrap sample
8. **Average effects** — `gen.ATE()`: integrated ATE/AME over data distribution
9. **Difference estimates** — Effect differences at moderator quantiles

### What is Excluded (per request.md)
- Fixed effects (felm/lfe) — not translated
- Instrumental variables (ivreg/AER) — not translated
- PCSE (panel-corrected SEs) — not translated
- Non-linear estimators (kernel, binning, GAM, DML, GRF) — not translated
- Plotting — not translated (users use matplotlib)
- Uniform CIs — not translated

## Target Repository Structure (New Package)

### Files to Create
```
statsclaw/example-probit/
├── pyproject.toml              # Package metadata, dependencies
├── README.md                   # Basic usage documentation
├── src/
│   └── interflex/
│       ├── __init__.py         # Public API exports
│       ├── core.py             # interflex() entry point, input validation
│       ├── linear.py           # interflex_linear() — main estimator
│       ├── effects.py          # Treatment effect computation (gen.general.TE equivalent)
│       ├── variance.py         # Delta method, simulation, bootstrap variance
│       ├── vcov.py             # Variance-covariance matrix estimation
│       └── average.py          # ATE/AME computation
└── tests/
    ├── __init__.py
    ├── test_basic.py           # Basic discrete/continuous treatment tests
    ├── test_variance.py        # Delta/simu/bootstrap variance tests
    ├── test_effects.py         # Treatment effect computation tests
    └── conftest.py             # Shared fixtures (simulated data)
```

## Risk Areas
1. **Numerical precision** — Delta method gradient computations must match R within tolerance
2. **Treatment type detection** — R uses factor-like detection; Python needs explicit handling
3. **Simulation variance** — Random draws must use same multivariate normal approach
4. **Bootstrap** — Block bootstrap for clustered data must match R's resampling strategy
5. **Named coefficient indexing** — R uses named vectors extensively; Python needs careful dict/array mapping

## Required Teammates
- **planner** — Produce spec.md (code spec) and test-spec.md (test spec)
- **builder** — Implement the Python package from spec.md (worktree)
- **tester** — Validate from test-spec.md
- **scriber** — ARCHITECTURE.md + documentation
- **reviewer** — Cross-pipeline quality gate
- **shipper** — Push to statsclaw/example-probit

## Profile
`python-package` — Python package with pytest validation

## Workflow
Workflow 2: Code + Ship
