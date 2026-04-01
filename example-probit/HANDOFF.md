# Handoff — example-probit

> Last updated: 2026-03-31 | Run: 20260331-interflex-py-224717

## Active Handoff Notes

1. **Factor Z_ref limitation**: When users provide `Z_ref` for categorical covariates, the current code defaults factor dummies to 0 rather than resolving the user-specified level to sum-coded values. This edge case needs enhancement if users routinely provide Z_ref for categorical covariates.

2. **No parallel bootstrap**: Bootstrap is sequential. For large datasets or many replicates, this may be slow. A future enhancement could add joblib parallelism.

3. **Excluded R features**: Fixed effects (felm/lfe), instrumental variables (ivreg/AER), PCSE, non-linear estimators (kernel, binning, GAM, DML, GRF), plotting, and uniform CIs are intentionally excluded. If any of these are needed, they would require significant new modules.

4. **Treatment group limit**: Maximum 9 discrete treatment groups, matching the R package constraint.

5. **Deprecated pandas API**: The code avoids the deprecated `pd.api.types.is_categorical_dtype()` by using `isinstance(dtype, pd.CategoricalDtype)` instead. This is compatible with pandas 2.x.

6. **Test count discrepancy**: Builder wrote 50 tests; tester expanded the test suite to 92 tests (adding more granular checks). All 92 pass.

7. **Build backend**: If the package is built with older setuptools that somehow has the `_legacy` backend, `pyproject.toml` would need to be updated. Current setup uses the standard `setuptools.build_meta`.
