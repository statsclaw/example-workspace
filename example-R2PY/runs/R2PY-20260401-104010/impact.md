# Impact Analysis: R-to-Python Translation of interflex Linear Estimator

## Source Files (R package xuyiqing/interflex)

| R File | Lines | Purpose | Translation Target |
|--------|-------|---------|-------------------|
| `R/interflex.R` | 1232 | Main entry point, input validation, routing | `interflex/core.py` — `interflex()` function |
| `R/linear.R` | 2249 | Full linear estimator: formula, fitting, variance, ATE | `interflex/linear.py` — core estimator |
| `R/plot.R` | 1560 | ggplot2 plotting for all estimators | `interflex/plotting.py` — matplotlib |
| `R/predict.R` | 730 | S3 predict method | `interflex/predict.py` |
| `R/vcluster.R` | 39 | Clustered variance-covariance | `interflex/vcov.py` |
| `R/uniform.R` | 40 | Uniform confidence intervals | `interflex/uniform.py` |
| `src/fastplm.cpp` | ~150 | FWL demeaning for FE | `interflex/fwl.py` — NumPy implementation |
| `src/iv_fastplm.cpp` | ~200 | FWL demeaning for IV+FE | `interflex/fwl.py` |

## Target Package Structure (statsclaw/example-R2PY)

```
interflex/
├── __init__.py
├── core.py           # interflex() entry, input validation
├── linear.py         # interflex_linear() — model fitting, formula construction
├── effects.py        # Treatment effects (TE/ME), ATE/AME, delta method
├── variance.py       # Simulation, bootstrap, delta variance paths
├── vcov.py           # vcovCluster, robust SE helpers
├── uniform.py        # Uniform CI calculations
├── fwl.py            # Frisch-Waugh-Lovell demeaning (replaces C++)
├── plotting.py       # matplotlib/seaborn plots
├── predict.py        # predict method
├── result.py         # InterflexResult class (replaces S3 "interflex" class)
└── _typing.py        # Type aliases
tests/
├── test_linear_discrete.py
├── test_linear_continuous.py
├── test_variance.py
├── test_vcov.py
├── test_fwl.py
├── test_ate.py
├── test_plotting.py
├── test_predict.py
└── conftest.py       # fixtures, synthetic data
pyproject.toml
README.md
```

## Key Translation Decisions

### R → Python Mapping

| R | Python |
|---|--------|
| `glm()` | `statsmodels.api.GLM` |
| `felm()` (lfe) | Custom FWL demeaning + OLS |
| `ivreg()` (AER) | `linearmodels.iv.IV2SLS` or custom 2SLS |
| `vcovHC(type="HC1")` | `statsmodels` HC1 or custom |
| `vcovCluster()` | Custom (translate directly) |
| `pcse()` | `linearmodels.PanelOLS` PCSE or custom |
| `rmvnorm()` | `numpy.random.multivariate_normal` |
| `glm.nb()` | `statsmodels.discrete.NegativeBinomial` |
| `ggplot2` | `matplotlib` + `seaborn` |
| S3 class "interflex" | `InterflexResult` dataclass |
| `dnorm()` / `pnorm()` | `scipy.stats.norm.pdf` / `.cdf` |

### Risk Areas

1. **Fixed effects via FWL**: The C++ `fastplm.cpp` iteratively demeans — must replicate convergence behavior in NumPy
2. **IV regression**: `felm` IV syntax is complex; need either `linearmodels` or custom 2SLS after FWL
3. **PCSE**: Panel-corrected SEs — may need `linearmodels` or custom implementation
4. **Negative binomial GLM**: `statsmodels` NB parameterization differs from R's `MASS::glm.nb`
5. **Numerical precision**: Delta method SE calculations must match R within tolerance

## Required Teammates

| Teammate | Task |
|----------|------|
| **planner** | Produce `spec.md` (Python package architecture, module-by-module translation spec), `test-spec.md` (test plan with synthetic data, numerical validation), `comprehension.md` |
| **builder** | Implement all Python modules from `spec.md` in worktree |
| **tester** | Validate from `test-spec.md`: unit tests, numerical matching |
| **scriber** | ARCHITECTURE.md, log entry, docs |
| **reviewer** | Cross-compare pipelines, verify numerical fidelity |
| **shipper** | Push to statsclaw/example-R2PY, sync workspace |

## Profile

`python-package` — see `profiles/python-package.md`

## Workflow

Workflow 2 (Code + Ship): `leader → planner → builder → tester → scriber → reviewer → shipper`
