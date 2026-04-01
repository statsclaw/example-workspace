# Simulation Specification (sim-spec.md)

```yaml
Request ID: probit-20260401-103705
Pipeline: Simulation (simulator only)
Profile: r-package
Package: exampleProbit
```

---

## 1. DGP Definition

### 1.1 Model Structure

Binary probit model with one covariate:

    Pr(y = 1 | x_1) = Phi(-1 + 0.5 * x_1)

Equivalently, using the latent variable formulation:

    y* = beta_0 + beta_1 * x_1 + eps,  eps ~ N(0, 1)
    y = 1(y* > 0)

### 1.2 True Parameters

| Parameter | Symbol | Value |
| --- | --- | --- |
| Intercept | beta_0 | -1 |
| Slope | beta_1 | 0.5 |
| True parameter vector | beta_true | (-1, 0.5)' |

### 1.3 Covariate Distribution

- x_1 ~ N(0, 1) independently across observations
- The design matrix X is N x 2: first column is a vector of ones (intercept), second column is x_1

### 1.4 Data Generation Procedure

For each replication r = 1, ..., R and sample size N:

1. Draw x_{1,i} ~ N(0, 1) for i = 1, ..., N
2. Form X = cbind(1, x_1)  (N x 2 matrix)
3. Compute probabilities p_i = Phi(-1 + 0.5 * x_{1,i})
4. Draw y_i ~ Bernoulli(p_i) for i = 1, ..., N

### 1.5 Sample Sizes

N in {200, 500, 1000, 5000}

---

## 2. Estimator Interface

The simulator treats each estimator as a black box. Call them via their R wrapper functions.

### 2.1 MLE

```r
fit_mle <- probit_mle(X, y, max_iter = 100, tol = 1e-8)
```

Extract:
- Point estimates: `fit_mle$coefficients` (numeric vector of length 2)
- Standard errors: `fit_mle$se` (numeric vector of length 2)
- 95% CI: `fit_mle$coefficients +/- 1.96 * fit_mle$se` (asymptotic normality)
- Convergence: `fit_mle$converged` (logical)

### 2.2 Gibbs Sampler

```r
fit_gibbs <- probit_gibbs(X, y, n_iter = 3500, burn_in = 500,
                          beta0 = rep(0, 2), Sigma0 = diag(100, 2))
```

Extract:
- Point estimates: `fit_gibbs$posterior_mean` (numeric vector of length 2)
- Standard errors: `fit_gibbs$posterior_sd` (numeric vector of length 2)
- 95% CI: 2.5% and 97.5% quantiles of `fit_gibbs$beta_draws` columns
  - `apply(fit_gibbs$beta_draws, 2, quantile, probs = c(0.025, 0.975))`

### 2.3 Metropolis-Hastings

```r
fit_mh <- probit_mh(X, y, n_iter = 10000, burn_in = 2000, scale = 1.0,
                    beta0 = rep(0, 2), Sigma0 = diag(100, 2))
```

Extract:
- Point estimates: `fit_mh$posterior_mean` (numeric vector of length 2)
- Standard errors: `fit_mh$posterior_sd` (numeric vector of length 2)
- 95% CI: 2.5% and 97.5% quantiles of `fit_mh$beta_draws` columns
- Acceptance rate: `fit_mh$acceptance_rate` (double)

---

## 3. Scenario Grid

### 3.1 Dimensions

| Dimension | Values | Count |
| --- | --- | --- |
| Sample size N | {200, 500, 1000, 5000} | 4 |
| Method | {MLE, Gibbs, MH} | 3 |

Total scenarios: 4 x 3 = 12

### 3.2 Replications

R = 500 replications per scenario

### 3.3 Total Simulation Runs

4 sample sizes x 3 methods x 500 replications = 6000 total estimator fits

### 3.4 Estimator Configurations

| Method | Parameter | Value |
| --- | --- | --- |
| MLE | max_iter | 100 |
| MLE | tol | 1e-8 |
| Gibbs | n_iter | 3500 |
| Gibbs | burn_in | 500 |
| MH | n_iter | 10000 |
| MH | burn_in | 2000 |
| MH | scale | 1.0 |
| Gibbs & MH | beta0 (prior mean) | rep(0, 2) |
| Gibbs & MH | Sigma0 (prior cov) | diag(100, 2) |

---

## 4. Performance Metrics

For each method m, each coefficient j in {0, 1} (intercept and slope), across R replications:

### 4.1 Bias

    Bias_j = (1/R) * sum_{r=1}^{R} (beta_hat_j^(r) - beta_true_j)

### 4.2 RMSE

    RMSE_j = sqrt((1/R) * sum_{r=1}^{R} (beta_hat_j^(r) - beta_true_j)^2)

### 4.3 Coverage

    Coverage_j = (1/R) * sum_{r=1}^{R} 1(beta_true_j in CI_95%^(r))

Where CI_95% is constructed as:
- MLE: beta_hat_j +/- 1.96 * SE_j (from asymptotic normality)
- Gibbs: [quantile(beta_draws[,j], 0.025), quantile(beta_draws[,j], 0.975)]
- MH: [quantile(beta_draws[,j], 0.025), quantile(beta_draws[,j], 0.975)]

### 4.4 Average Time

    Time_m = (1/R) * sum_{r=1}^{R} t_m^(r)  (seconds)

Time the entire estimator call for each replication using `system.time()` or `proc.time()`. Report wall-clock time (elapsed).

### 4.5 MH Acceptance Rate

    AcceptRate = (1/R) * sum_{r=1}^{R} acceptance_rate^(r)

Only for MH method.

---

## 5. Acceptance Criteria

| ID | Criterion | Threshold |
| --- | --- | --- |
| C1 | MLE coefficients match glm within tolerance | max abs diff < 0.05 per replication (median across replications) |
| C2 | Gibbs and MH posterior means within tolerance of MLE (N >= 500) | abs(Bias_Bayes - Bias_MLE) < 0.1 per coefficient |
| C3 | MH acceptance rate | Mean across replications in [0.20, 0.50] for each N |
| C4 | 95% CI coverage | In [0.90, 0.99] for all methods, coefficients, and N |
| C5 | MLE speed advantage over Gibbs | Time_Gibbs / Time_MLE >= 5.0 for each N |

---

## 6. Seed Strategy

### 6.1 Master Seed

Use a single master seed at the start of the simulation script:

```r
set.seed(2026)
```

### 6.2 Per-Replication Seeds

For reproducibility, pre-generate R seeds from the master RNG:

```r
set.seed(2026)
seeds <- sample.int(.Machine$integer.max, R)
```

For each replication r, set `set.seed(seeds[r])` before generating data. This ensures:
- Results are reproducible given the master seed
- Each replication uses independent random streams
- The same seed sequence is used across all N and methods for a given replication

### 6.3 Simulation Loop Structure

```
for each N in {200, 500, 1000, 5000}:
    for each r in 1:R:
        set.seed(seeds[r])
        Generate data (X, y) with this N
        For each method (MLE, Gibbs, MH):
            Time the estimator call
            Extract point estimates, CI bounds, acceptance rate (MH only)
            Record convergence status (MLE only)
        Store results for this (N, r) combination
```

**Important**: The same (X, y) data is used for all three methods within the same (N, r) combination. This ensures fair comparison.

---

## 7. Output Format

### 7.1 Results Table Structure

The simulation output should be a data frame (or equivalent) with columns:

| Column | Type | Description |
| --- | --- | --- |
| N | int | Sample size |
| method | char | "MLE", "Gibbs", or "MH" |
| param | char | "beta_0" or "beta_1" |
| bias | double | Bias for this coefficient |
| rmse | double | RMSE for this coefficient |
| coverage | double | 95% CI coverage rate |
| time | double | Average time in seconds (same for both params within a method-N combo) |
| accept_rate | double | Mean acceptance rate (MH only; NA for MLE and Gibbs) |

This gives 4 (N) x 3 (methods) x 2 (parameters) = 24 rows in the summary table.

### 7.2 Raw Results Storage

For each (N, method, replication), store:
- beta_hat (vector of length 2)
- CI_lower (vector of length 2)
- CI_upper (vector of length 2)
- time (scalar)
- acceptance_rate (scalar, MH only)
- converged (logical, MLE only)

Save raw results as an RDS file: `inst/simulation/simulation_results.rds`

Save the summary table in the run directory as part of `simulation.md`.

### 7.3 Summary Table Format

The summary table in `simulation.md` should look like:

```
| N    | Method | Param  | Bias   | RMSE   | Coverage | Time(s) | Accept |
| ---- | ------ | ------ | ------ | ------ | -------- | ------- | ------ |
| 200  | MLE    | beta_0 | ...    | ...    | ...      | ...     | NA     |
| 200  | MLE    | beta_1 | ...    | ...    | ...      | ...     | NA     |
| 200  | Gibbs  | beta_0 | ...    | ...    | ...      | ...     | NA     |
| ...  | ...    | ...    | ...    | ...    | ...      | ...     | ...    |
| 5000 | MH     | beta_1 | ...    | ...    | ...      | ...     | ...    |
```

---

## 8. Implementation Location

### 8.1 Simulation Script

Place the simulation script at: `inst/simulation/run_simulation.R`

This script should:
1. Load the `exampleProbit` package
2. Run the full simulation grid
3. Compute all metrics
4. Save raw results to `inst/simulation/simulation_results.rds`
5. Print the summary table

### 8.2 R Wrapper Function (Optional)

Optionally provide an R function `simulate_probit()` in `R/simulate_probit.R` that runs the simulation and returns results. If provided:
- @param N_vec: numeric vector of sample sizes (default c(200, 500, 1000, 5000))
- @param R: number of replications (default 500)
- @param seed: master seed (default 2026)
- @return: list with raw_results (data frame) and summary_table (data frame)
- Wrap in `\donttest{}` in examples since it is computationally expensive

### 8.3 Simulation Artifact

After running the simulation, write `simulation.md` to the run directory with:
- The summary table
- Any diagnostic notes (convergence failures, timing anomalies)
- Acceptance criteria validation results (C1-C5 pass/fail with evidence)

---

## 9. Computational Notes

- The simulation will perform 6000 estimator fits total. MH with 10000 iterations at N=5000 is the most expensive.
- Estimate total runtime: MLE is fast (< 0.01s per fit at N=5000), Gibbs moderate (0.1-0.5s), MH most expensive (0.5-2s). Total: approximately 30-60 minutes.
- Do NOT parallelize in the simulation script (CRAN compliance: max 2 cores in tests/examples). Run sequentially.
- Use `message()` to report progress (e.g., every 100 replications) so the user knows the simulation is running.
- If MLE does not converge for a replication, record NA for that replication's MLE results and note the failure count.
