# Mailbox

## Handoff: Planner -> Builder (spec.md)

Builder should implement the full `interflex` Python package as specified in `spec.md`. The package has 11 modules. Start with `_typing.py` and `result.py` (no dependencies), then `fwl.py` and `uniform.py` (standalone algorithms), then `vcov.py`, `effects.py`, and `variance.py` (core computation), then `linear.py` and `core.py` (integration), and finally `plotting.py` and `predict.py`. Key risk areas: (1) FWL demeaning must exactly replicate the C++ convergence behavior; (2) negative binomial parameterization differs between R and statsmodels; (3) the delta method gradient vectors for non-linear models (logit, probit, poisson) are complex and must be implemented precisely per the formulas in spec.md Section 4.5-4.6. The package structure, pyproject.toml, and all function signatures are fully specified.

## Handoff: Planner -> Tester (test-spec.md)

Tester should create tests as specified in `test-spec.md`. Nine synthetic data recipes (A-I) provide self-contained test data with known DGP parameters. The test matrix covers 16 method/vartype/vcov_type/treat_type combinations. Priority: focus first on HIGH priority tests (linear discrete delta robust, linear continuous, FE, clustered SE, input validation, result structure). Tolerance levels are specified per quantity. Key property-based invariants to verify: TE linearity, prediction consistency, difference additivity, vcov PSD, FE prediction=0. Validation command: `pytest tests/ -v --tb=short`.
