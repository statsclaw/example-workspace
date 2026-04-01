# Documentation Report (docs.md)

```yaml
Request ID: probit-20260401-103705
Pipeline: Scriber
Package: exampleProbit
```

---

## Documentation Status

All exported functions are fully documented with roxygen2. Documentation was generated automatically during the build process via `roxygen2::roxygenize()`.

## Exported Functions

### `probit_mle(X, y, max_iter = 100L, tol = 1e-8)`

**File**: `R/probit_mle.R` | **Help page**: `man/probit_mle.Rd`

Estimates the probit model by maximum likelihood using Newton-Raphson in C++.

Parameters:
- `X`: Numeric matrix (N x k) of covariates; must include intercept column if desired
- `y`: Numeric vector of length N with binary outcomes (0 or 1)
- `max_iter`: Maximum Newton-Raphson iterations (default 100)
- `tol`: Convergence tolerance on step norm (default 1e-8)

Returns a list with: `coefficients` (numeric vector), `vcov` (k x k matrix), `se` (numeric vector), `loglik` (scalar), `iterations` (integer), `converged` (logical).

Example is unwrapped (runs quickly).

### `probit_gibbs(X, y, n_iter = 3500L, burn_in = 500L, beta0 = NULL, Sigma0 = NULL)`

**File**: `R/probit_gibbs.R` | **Help page**: `man/probit_gibbs.Rd`

Estimates the probit model using the Albert-Chib Gibbs sampler with data augmentation in C++.

Parameters:
- `X`, `y`: Same as MLE
- `n_iter`: Total MCMC iterations (default 3500)
- `burn_in`: Burn-in iterations to discard (default 500)
- `beta0`: Optional prior mean vector (default: zeros)
- `Sigma0`: Optional prior covariance matrix (default: 100 * I)

Returns a list with: `beta_draws` (matrix, (n_iter - burn_in) x k), `posterior_mean` (numeric vector), `posterior_sd` (numeric vector).

Example wrapped in `\donttest{}` (MCMC is slow).

### `probit_mh(X, y, n_iter = 10000L, burn_in = 2000L, scale = 2.4, beta0 = NULL, Sigma0 = NULL, init = NULL)`

**File**: `R/probit_mh.R` | **Help page**: `man/probit_mh.Rd`

Estimates the probit model using random-walk Metropolis-Hastings in C++. Proposal distribution is multivariate normal with covariance proportional to (X'X)^{-1}.

Parameters:
- `X`, `y`: Same as MLE
- `n_iter`: Total MCMC iterations (default 10000)
- `burn_in`: Burn-in iterations to discard (default 2000)
- `scale`: Proposal scale parameter (default 2.4); tunes acceptance rate toward 20-50 percent
- `beta0`, `Sigma0`: Optional prior parameters
- `init`: Optional initial beta vector (default: MLE estimate)

Returns a list with: `beta_draws` (matrix), `posterior_mean` (numeric vector), `posterior_sd` (numeric vector), `acceptance_rate` (scalar in [0, 1]).

Example wrapped in `\donttest{}` (MCMC is slow).

### `run_probit_simulation(N_vec, R, seed, verbose)`

**File**: `R/simulate_probit.R` | **Help page**: `man/run_probit_simulation.Rd` (generated)

Runs a Monte Carlo simulation comparing all three estimation methods across a grid of sample sizes.

Parameters:
- `N_vec`: Integer vector of sample sizes (default: c(200, 500, 1000, 5000))
- `R`: Number of replications (default: 500)
- `seed`: Master seed (default: 2026)
- `verbose`: Logical; print progress (default: TRUE)

Returns a list with: `raw_results` (list of per-replication data) and `summary_table` (data frame with bias, RMSE, coverage, time, acceptance rate).

Example wrapped in `\donttest{}` (computationally expensive).

## Internal Functions

- `.compute_sim_metrics()`: Internal helper in `R/simulate_probit.R` that computes bias, RMSE, coverage, time, and acceptance rate for one (N, method, parameter) combination. Not exported. Documented with `@keywords internal`.

## Package-Level Documentation

- `R/exampleProbit-package.R`: Contains `@useDynLib exampleProbit, .registration = TRUE` and `@importFrom Rcpp sourceCpp` for proper shared library registration.

## Generated Documentation Files

| File | Status | Notes |
| --- | --- | --- |
| `man/probit_mle.Rd` | generated | MLE help page, all params documented |
| `man/probit_gibbs.Rd` | generated | Gibbs help page, all params documented |
| `man/probit_mh.Rd` | generated | MH help page, scale default = 2.4, "20-50 percent" text |
| `man/exampleProbit-package.Rd` | generated | Package overview |
| `NAMESPACE` | generated | Exports: probit_mle, probit_gibbs, probit_mh, run_probit_simulation |

## CRAN Compliance

- All examples present and runnable
- MCMC examples wrapped in `\donttest{}`
- No hardcoded `set.seed()` in exported functions (only in examples/tests)
- `message()` used for output, not `print()` or `cat()`
- `Authors@R` format in DESCRIPTION
- Title in Title Case
- Description is 2+ sentences
- TRUE/FALSE used throughout (no T/F)

## Documentation Generation Commands

To regenerate documentation after any roxygen2 changes:

```r
roxygen2::roxygenize(".")
```

This regenerates `NAMESPACE` and all `man/*.Rd` files.

## ARCHITECTURE.md

Produced and placed in target repo root (`ARCHITECTURE.md`). Also copied to run directory for reviewer verification. Contains:
- Module structure diagram (Mermaid) with R API, C++ core, utilities, simulation, and glue layers
- Function call graph (Mermaid) tracing from R wrappers through C++ to shared utilities
- Data flow diagram (Mermaid) showing MLE, Gibbs, and MH estimation paths
- Module and function reference tables
- Architectural patterns and notes

## Deferred Items

None. All exported functions are fully documented. All examples are self-contained and runnable.
