# Comprehension Record

```yaml
RequestID: REQ-20260331-223247
Timestamp: 2026-03-31T22:45:00
HOLDRoundsUsed: 0
Verdict: FULLY UNDERSTOOD
```

## Input Materials Read

| Material | Type | Contents |
|----------|------|----------|
| `probit_spec.pdf` | PDF (5 pages) | Full mathematical specification: model, 3 estimators (MLE, Gibbs, MH), shared utilities, Monte Carlo design |
| `request.md` | Markdown | Scope, acceptance criteria C1-C7, ship instructions |
| `impact.md` | Markdown | File inventory for greenfield R package, risk areas, teammate assignments |
| `r-package.md` | Profile | R package conventions, CRAN checklist, validation commands |

## Core Requirement (Restated from Memory)

Build a greenfield R package `exampleProbit` that implements three probit estimation methods in C++ via Rcpp/RcppArmadillo: (1) MLE via Newton-Raphson with analytic gradient and Hessian, (2) Bayesian Gibbs sampler using the Albert-Chib data augmentation scheme with truncated normal latent variables, and (3) random-walk Metropolis-Hastings MCMC with log-posterior evaluation. The package includes shared numerical utilities for safe log-Phi evaluation and truncated normal sampling. A Monte Carlo simulation study compares all three methods across four sample sizes (200, 500, 1000, 5000) with 500 replications, measuring bias, RMSE, coverage, and computation time. The DGP is a simple probit with intercept and one covariate.

## Formulas Restated and Verified

### Model (Section 1)

Latent variable formulation: y*_i = x'_i beta + eps_i, eps_i ~ N(0,1), y_i = 1(y*_i > 0).
Observation probability: Pr(y_i = 1 | x_i) = Phi(x'_i beta).
Notation: N = sample size, k = number of covariates (including intercept), X in R^{N x k} (row i is x'_i), phi(.) = standard normal PDF, Phi(.) = standard normal CDF.

**Verified against PDF**: Matches equation (1) exactly.

### MLE (Section 2)

Log-likelihood (eq 2): l(beta) = sum_i [y_i log Phi(x'_i beta) + (1-y_i) log(1 - Phi(x'_i beta))]

Auxiliary quantities: q_i = 2y_i - 1 in {-1,+1}, eta_i = x'_i beta, lambda_i = phi(q_i eta_i) / Phi(q_i eta_i) (inverse Mills ratio).

Gradient (eq 3): nabla l = X'w, where w_i = q_i lambda_i.
Hessian (eq 4): nabla^2 l = -X'DX, where D = diag(d_1,...,d_N), d_i = lambda_i(lambda_i + q_i eta_i) > 0.

Newton-Raphson update (Algorithm 1): beta^(t) = beta^(t-1) + (X'DX)^{-1} X'w. Stop when ||beta^(t) - beta^(t-1)|| < 1e-8. Max 100 iterations.

Output: beta_hat, SE_j = sqrt([(X'DX)^{-1}]_{jj}), l(beta_hat).

Numerical stability: Use R::pnorm(x, 0, 1, 1, 1) for log Phi(x). Solve via arma::solve, not explicit inversion.

C++ signature: `Rcpp::List probit_mle(const arma::mat& X, const arma::vec& y, int max_iter = 100, double tol = 1e-8)`
Returns: coefficients (vec), vcov (mat), se (vec), loglik (double).

**Verified against PDF**: All equations match. The Newton step uses the negative Hessian as the information matrix, consistent with IRLS for probit. d_i > 0 is guaranteed because lambda_i > 0 and Phi(q_i eta_i) > 0.

### Gibbs Sampler (Section 3)

Prior: beta ~ N(beta_0, Sigma_0). Default: beta_0 = 0, Sigma_0 = 100*I_k.

Latent variable: y* = (y*_1, ..., y*_N)' where y*_i ~ N(x'_i beta, 1).

Step 1 — Sample beta | X, y*:
  Posterior: beta | X, y* ~ N(beta_tilde, Sigma_tilde)
  Sigma_tilde = (Sigma_0^{-1} + X'X)^{-1}   (eq 5-6)
  beta_tilde = Sigma_tilde (Sigma_0^{-1} beta_0 + X'y*)

Step 2 — Sample y*_i | beta, y_i:
  If y_i = 1: y*_i ~ TN(x'_i beta, 1; 0, +inf)
  If y_i = 0: y*_i ~ TN(x'_i beta, 1; -inf, 0)   (eq 7)

Truncated normal sampling via inverse CDF (eq 8):
  y*_i = mu_i + Phi^{-1}(Phi(a - mu_i) + u * [Phi(b - mu_i) - Phi(a - mu_i)]), u ~ Unif(0,1)
  where mu_i = x'_i beta and (a,b) depends on y_i.

Key optimization: Sigma_tilde and its Cholesky L depend only on X and the prior — precompute once outside the loop.

Algorithm 2: Initialize beta^(0) = 0. For t = 1,...,T: sample y* (Step 2), compute beta_tilde (Step 1), draw beta^(t) = beta_tilde + L*z where z ~ N(0, I_k). Discard first B draws as burn-in.

C++ signature: `Rcpp::List probit_gibbs(const arma::mat& X, const arma::vec& y, int n_iter = 3500, int burn_in = 500, Nullable<arma::vec> beta0, Nullable<arma::mat> Sigma0)`
Returns: beta_draws (mat), posterior_mean (vec), posterior_sd (vec).

**Verified against PDF**: Matches equations (5)-(8) and Algorithm 2. The derivation of the posterior is standard conjugate normal-normal.

### Metropolis-Hastings (Section 4)

Log-posterior (eq 9): log p(beta | X, y) = l(beta) - 0.5 * (beta - beta_0)' Sigma_0^{-1} (beta - beta_0) + const.

Proposal: beta' = beta^(t) + eps, eps ~ N(0, s^2 (X'X)^{-1}).

Accept probability: alpha = min(1, exp[log p(beta' | .) - log p(beta^(t) | .)]).

Algorithm 3:
1. Initialize beta^(0) from MLE. Precompute proposal Cholesky L_C = chol(s^2 (X'X)^{-1}).
2. For t = 1,...,T: propose beta' = beta^(t-1) + L_C * z, z ~ N(0,I_k). Compute log ratio r = log p(beta'|.) - log p(beta^(t-1)|.). If log u < log r (u ~ Unif), accept; else keep.
3. Discard burn-in. Output: posterior draws, acceptance rate (target 25-40%).

C++ signature: `Rcpp::List probit_mh(const arma::mat& X, const arma::vec& y, int n_iter = 10000, int burn_in = 2000, double scale = 1.0, Nullable<arma::vec> beta0, Nullable<arma::mat> Sigma0, Nullable<arma::vec> init)`
Returns: beta_draws (mat), posterior_mean (vec), posterior_sd (vec), acceptance_rate (double).

**Verified against PDF**: Matches equation (9) and Algorithm 3. The proposal covariance s^2(X'X)^{-1} scales with the Fisher information, which is standard practice.

### Shared Utilities (Section 5)

- `safe_log_Phi(x)`: Returns log Phi(x) via R::pnorm(x, 0, 1, 1, 1) — log-space computation avoids underflow.
- `safe_Phi(x)`: Returns Phi(x) via R::pnorm(x, 0, 1, 1, 0).
- `rtruncnorm(mu, lo, hi)`: Inverse-CDF truncated normal sampler per equation (8).

**Verified**: These are standard numerical computing best practices for probit models.

### Monte Carlo Design (Section 6)

- DGP: Pr(y=1|x1) = Phi(-1 + 0.5*x1), x1 ~ N(0,1). True beta_0 = (-1, 0.5)'.
- Design matrix: X = [1, x1] (intercept + one covariate), so k=2.
- Sample sizes: N in {200, 500, 1000, 5000}.
- Replications: R = 500 per scenario.
- MLE settings: max_iter=100, tol=1e-8.
- Gibbs settings: n_iter=3500, burn_in=500.
- MH settings: n_iter=10000, burn_in=2000, scale=1.0.
- Prior (Gibbs & MH): beta_0 = 0, Sigma_0 = 100*I_2.

Metrics (per method, per beta_j, across R replications):
- Bias_j = (1/R) sum_r (beta_hat_j^(r) - beta_{0,j})      (eq 10)
- RMSE_j = sqrt((1/R) sum_r (beta_hat_j^(r) - beta_{0,j})^2)   (eq 10)
- Coverage_j = (1/R) sum_r 1(beta_{0,j} in CI^(r)_95%)     (eq 11)
- Time = (1/R) sum_r t^(r) (seconds)                        (eq 11)

CI construction:
- MLE: beta_hat_j +/- 1.96 * SE_j (asymptotic normality).
- Gibbs/MH: 2.5% and 97.5% quantiles of posterior draws.

**Verified against PDF**: All metrics and design parameters match Section 6.

## Comprehension Self-Test

### 1. Can I restate the core requirement without looking at the source?
Yes — done above in "Core Requirement" section.

### 2. Can I write out every formula from memory?
Yes — all formulas restated above. Cross-checked against the PDF. No discrepancies found.

### 3. Are there any symbols I cannot precisely define?
No. All symbols are defined:
- N: sample size (scalar, positive integer)
- k: number of covariates including intercept (scalar, positive integer; k=2 for simulation)
- X: N x k design matrix (real-valued)
- y: N x 1 binary response vector ({0,1})
- y*: N x 1 latent variable vector (real-valued)
- beta: k x 1 coefficient vector
- beta_0: k x 1 prior mean (default: zero vector)
- Sigma_0: k x k prior covariance (default: 100*I_k)
- Phi(.): standard normal CDF
- phi(.): standard normal PDF
- q_i: sign transform 2y_i - 1 in {-1,+1}
- eta_i: linear predictor x'_i beta (scalar)
- lambda_i: inverse Mills ratio phi(q_i eta_i)/Phi(q_i eta_i) (scalar, positive)
- d_i: Hessian diagonal element lambda_i(lambda_i + q_i eta_i) (scalar, positive)
- w_i: gradient weight q_i lambda_i (scalar)
- D: N x N diagonal matrix diag(d_1,...,d_N)
- Sigma_tilde: k x k posterior covariance for beta in Gibbs
- beta_tilde: k x 1 posterior mean for beta in Gibbs
- L: k x k lower-triangular Cholesky of Sigma_tilde
- L_C: k x k lower-triangular Cholesky of proposal covariance s^2(X'X)^{-1}
- s: MH proposal scale (scalar, default 1.0)
- R: number of Monte Carlo replications (500)
- T: total MCMC iterations
- B: burn-in count

### 4. Are there steps requiring judgment calls not in the source?
No. The spec is complete. Every algorithm step, convergence criterion, default value, and interface is fully specified.

### 5. Can I explain the mathematical intuition?
Yes:
- **MLE**: The probit log-likelihood is globally concave (the Hessian is negative definite since d_i > 0 for all i), so Newton-Raphson converges to the unique global maximum. The q_i reparametrization simplifies the gradient/Hessian to clean matrix expressions.
- **Gibbs (Albert-Chib)**: Introduces latent continuous y* so that, conditional on y*, beta has a conjugate normal posterior (normal-normal). Conditional on beta, each y*_i is an independent truncated normal. Alternating these two steps defines a valid Gibbs sampler that converges to the joint posterior.
- **MH**: Uses the MLE as a warm start and the inverse Fisher information (X'X)^{-1} scaled by s^2 as the proposal covariance. This matches the curvature of the posterior, giving reasonable acceptance rates (25-40%) without manual tuning of each dimension.
- **Monte Carlo**: By repeating estimation across many datasets drawn from a known DGP, we can empirically verify frequentist properties (bias, RMSE, coverage) and compare computational efficiency across methods.

### 6. Implicit assumptions?
- The design matrix X must have full column rank (k <= N) for (X'X)^{-1} and (X'DX)^{-1} to exist.
- For MLE, perfect separation (all y_i = 1 or all y_i = 0) would cause non-convergence; the simulation DGP with N >= 200 makes this astronomically unlikely.
- The Gibbs sampler assumes the prior precision Sigma_0^{-1} is well-defined (Sigma_0 must be positive definite).
- The inverse-CDF truncated normal sampler assumes Phi(b-mu) - Phi(a-mu) > 0, which holds as long as the truncation interval is non-degenerate.
- MCMC convergence is assumed after the specified burn-in periods (not formally diagnosed).

All implicit assumptions are standard and non-controversial for this problem setting. No user clarification needed.

## Comprehension Verdict

**FULLY UNDERSTOOD** — All formulas verified, all symbols defined, all algorithms specified without ambiguity, all metrics and acceptance criteria clear. No HOLD needed. Proceeding to spec production.
