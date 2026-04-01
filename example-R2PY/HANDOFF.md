# Handoff Notes — example-R2PY

> Last updated: 2026-04-01 | Run: R2PY-20260401-104010

1. **pyproject.toml build-backend**: Was `setuptools.backends._legacy:_Backend` (invalid). Fixed to `setuptools.build_meta` during shipping. Verify `pip install -e .` works.

2. **Single treatment arm error (E1)**: When only one treatment value is present, the package raises `UnboundLocalError` deep in `variance.py` instead of `ValueError` at input validation. Fix: add a check in `core.py` after treatment type detection that `len(unique_D) >= 2` for discrete treatment.

3. **Negative binomial cross-validation**: statsmodels NB2 uses `alpha = 1/theta` parameterization which differs from R's `MASS::glm.nb()`. Full numerical cross-validation against the R package has not been done for this method.

4. **Matplotlib figure leak**: Running many tests produces `RuntimeWarning: More than 20 figures opened`. Package should call `plt.close()` after saving/returning figures, or use `plt.ioff()` during testing.

5. **Parallel bootstrap overhead**: `ProcessPoolExecutor` serialization overhead may make parallel bootstrap slower than sequential for small datasets. Consider adding a heuristic to skip parallelization below a threshold (e.g., n < 1000 or nboots < 50).

6. **PCSE performance**: The current O(T * N^2) implementation is correct but could be optimized for large panels using vectorized pairwise operations.

7. **L-kurtosis diagnostic**: Not implemented. The R version computes this as a diagnostic statistic but it does not affect numerical results.
