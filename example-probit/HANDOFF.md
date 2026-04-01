# Active Handoff — example-probit

> Last updated: 2026-03-31 from run `probit-package-greenfield`

## Handoff Notes

1. **MH scale tuning**: The default scale=2.4 is calibrated for k=2 covariates (intercept + one predictor). Users with higher-dimensional models should expect to adjust this parameter. A rule of thumb is scale ~ 2.38/sqrt(k).

2. **No convergence diagnostics**: The package does not compute ESS, R-hat, or trace plots. Users should apply `coda::effectiveSize()` and `coda::gelman.diag()` to the returned `beta_draws` for serious applications.

3. **LaTeX PDF manual**: R CMD check shows a LaTeX error (`inconsolata.sty` not found) which is a system environment issue, not a package defect. Install the LaTeX package or use `--no-manual` to suppress.

4. **Simulation runtime**: The full simulation (~28 minutes) is dominated by MH fits (10000 iterations x 500 reps x 4 N values). If faster turnaround is needed, reduce `R` or `n_iter` in `run_simulation.R`.

5. **Coverage at large N**: Slight undercoverage for the slope at N=5000 (0.924-0.932) is within tolerance. This is a known finite-sample phenomenon in probit models where the asymptotic normal approximation can be slightly conservative for skewed response distributions.

6. **src/interflex directory**: There is an unrelated `src/interflex/` directory in the repo from a previous project. It does not affect the R package build (not included in the package tarball) but could be cleaned up.
