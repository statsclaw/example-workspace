# Comprehension Record

```yaml
Request ID: probit-20260401-103705
Planner: planner
Timestamp: 2026-04-01
HOLD rounds used: 0
Verdict: FULLY UNDERSTOOD
```

## Input Materials Read

| Material | Type | Contents |
| --- | --- | --- |
| `probit_spec.pdf` (5 pages) | PDF | Complete implementation specification: model notation, three estimation methods (MLE, Gibbs, MH) with full mathematical derivations, C++ interface signatures, shared utilities, Monte Carlo simulation design, metrics, and acceptance criteria |
| `request.md` | Markdown | Scope (exampleProbit R package), acceptance criteria C1-C7, ship instructions |
| `impact.md` | Markdown | File manifest, risk areas, teammate assignments, workflow 11 selection |

All uploaded files have been read completely.

## Restated Core Requirement

Build an R package called `exampleProbit` that implements three probit model estimators in C++ via Rcpp/RcppArmadillo: (1) Maximum Likelihood via Newton-Raphson, (2) Bayesian Gibbs sampler using Albert-Chib data augmentation, and (3) Random-walk Metropolis-Hastings MCMC. The package must include shared numerical utilities for stable log-CDF and truncated normal sampling, R wrapper functions with roxygen2 documentation, and a Monte Carlo simulation study comparing all three methods across four sample sizes on bias, RMSE, 95% CI coverage, and computational speed.

## All Formulas Restated and Verified

### Model (Eq. 1)

Latent variable formulation for i = 1, ..., N:

- y*_i = x'_i * beta + eps_i, where eps_i ~ N(0,1)
- y_i = 1(y*_i > 0)
- This yields Pr(y_i = 1 | x_i) = Phi(x'_i * beta)

Notation: N = sample size, k = number of covariates (including intercept), X in R^{N x k} = design matrix (row i is x'_i), phi(.) = standard normal PDF, Phi(.) = standard normal CDF.

### MLE (Eqs. 2-4)

Log-likelihood (Eq. 2):

    l(beta) = sum_{i=1}^{N} [ y_i * log Phi(x'_i * beta) + (1 - y_i) * log(1 - Phi(x'_i * beta)) ]

Define q_i = 2*y_i - 1 in {-1, +1}, eta_i = x'_i * beta, lambda_i = phi(q_i * eta_i) / Phi(q_i * eta_i) (inverse Mills ratio).

Gradient (Eq. 3): grad l = X' * w, where w_i = q_i * lambda_i

Hessian (Eq. 4): Hessian l = -X' * D * X, where d_i = lambda_i * (lambda_i + q_i * eta_i) > 0, D = diag(d_1, ..., d_N)

Newton-Raphson update: beta^(t) = beta^(t-1) + (X' D X)^{-1} X' w

Convergence: ||beta^(t) - beta^(t-1)|| < 1e-8

Output: beta_hat, SE_j = sqrt([(X' D X)^{-1}]_{jj}), l(beta_hat)

Verified against PDF: Matches exactly. The Hessian is negative definite since d_i > 0, guaranteeing concavity.

### Gibbs Sampler (Eqs. 5-8)

Prior: beta ~ N(beta_0, Sigma_0). Default: beta_0 = 0, Sigma_0 = 100 * I_k.

Step 1 - Sample beta (Eqs. 5-6):

    beta | X, y* ~ N(beta_tilde, Sigma_tilde)
    Sigma_tilde = (Sigma_0^{-1} + X'X)^{-1}
    beta_tilde = Sigma_tilde * (Sigma_0^{-1} * beta_0 + X' * y*)

Derivation verified: Combining the normal likelihood p(y* | beta) with the normal prior via Bayes' theorem and completing the square yields the precision matrix Sigma_0^{-1} + X'X and the precision-weighted mean Sigma_0^{-1} * beta_0 + X' * y*.

Step 2 - Sample y*_i (Eq. 7):

    y*_i | (y_i = 1, beta) ~ TN(x'_i * beta, 1; 0, +inf)
    y*_i | (y_i = 0, beta) ~ TN(x'_i * beta, 1; -inf, 0)

Truncated normal sampling via inverse CDF (Eq. 8):

    y*_i = mu_i + Phi^{-1}(Phi(a - mu_i) + u * [Phi(b - mu_i) - Phi(a - mu_i)])
    where mu_i = x'_i * beta, u ~ Unif(0,1), (a,b) = (0, +inf) or (-inf, 0)

Key optimization: Sigma_tilde and its Cholesky L depend only on X and the prior -- precompute once outside the loop.

Sampling beta: beta^(t) = beta_tilde + L * z, where z ~ N(0, I_k).

Verified against PDF: Matches exactly.

### Metropolis-Hastings (Eq. 9)

Log-posterior (Eq. 9):

    log p(beta | X, y) = l(beta) - 0.5 * (beta - beta_0)' * Sigma_0^{-1} * (beta - beta_0)

where l(beta) is the probit log-likelihood from Eq. 2.

Random walk proposal: beta' = beta^(t) + eps, eps ~ N(0, s^2 * (X'X)^{-1})

Accept with probability alpha = min(1, exp[log p(beta' | .) - log p(beta^(t) | .)])

Implementation: Precompute proposal Cholesky L_C = chol(s^2 * (X'X)^{-1}). Propose beta' = beta^(t-1) + L_C * z, z ~ N(0, I_k). Accept/reject on log scale.

Warm start: Initialize beta^(0) from MLE.

Target acceptance rate: 25-40%.

Verified against PDF: Matches exactly.

### Simulation Metrics (Eqs. 10-11)

Bias_j = (1/R) * sum_{r=1}^{R} (beta_hat_j^(r) - beta_{0,j})

RMSE_j = sqrt((1/R) * sum_{r=1}^{R} (beta_hat_j^(r) - beta_{0,j})^2)

Coverage_j = (1/R) * sum_{r=1}^{R} 1(beta_{0,j} in CI_95%^(r))

Time = (1/R) * sum_{r=1}^{R} t^(r) (seconds)

MLE CI: beta_hat_j +/- 1.96 * SE_j (asymptotic normality)
Gibbs/MH CI: 2.5% and 97.5% quantiles of posterior draws

Verified against PDF: Matches exactly.

## Undefined Symbols or Concepts

None. All symbols are fully defined in the specification.

## Judgment Calls Required

None. The specification is complete and unambiguous for all three methods, the simulation design, and the acceptance criteria.

## Mathematical/Statistical Intuition

**Why the probit model works**: The latent variable y* = x'beta + eps with eps ~ N(0,1) generates a binary outcome y = 1(y* > 0). The probability Pr(y=1|x) = Phi(x'beta) follows because Pr(eps > -x'beta) = Phi(x'beta) by symmetry of the normal distribution.

**Why MLE Newton-Raphson works**: The probit log-likelihood is globally concave (the Hessian -X'DX is negative definite since d_i > 0). Newton-Raphson converges quadratically to the unique global maximum from any starting point.

**Why Albert-Chib Gibbs works**: By introducing the latent y*, the model becomes a linear regression y* = X*beta + eps with known variance. The conditional posteriors are conjugate: beta|y* is normal, y*|beta is truncated normal. This makes both conditionals easy to sample from, and the Gibbs sampler converges to the joint posterior.

**Why MH works**: The random-walk proposal centered at the current state with covariance scaled by (X'X)^{-1} adapts to the curvature of the posterior. The scale parameter s controls the acceptance rate -- too large means low acceptance (proposals far from mode), too small means slow exploration (proposals close to current state). The 25-40% target balances exploration and acceptance.

**Why the proposal covariance uses (X'X)^{-1}**: This is approximately proportional to the posterior covariance (from asymptotic normality of the MLE), so proposals are shaped to follow the posterior geometry, yielding better mixing than an isotropic proposal.

## Implicit Assumptions

1. The design matrix X includes an intercept column (k >= 2 for the simulation DGP which has intercept + one covariate).
2. X'X is invertible (full column rank), required for the MLE variance-covariance matrix, the Gibbs posterior covariance, and the MH proposal covariance.
3. The prior covariance Sigma_0 is positive definite (100*I satisfies this).
4. The probit model is correctly specified in the simulation DGP (data are actually generated from a probit model).
5. For the truncated normal sampling, Phi(b - mu) - Phi(a - mu) > 0 always holds since the truncation region has positive probability under the normal distribution.

All assumptions are standard and supported by the specification.

## Comprehension Verdict

**FULLY UNDERSTOOD**

All formulas have been restated and verified against the source. All symbols are defined. All algorithms are unambiguous. No judgment calls are required. The specification is complete for producing all three pipeline specs.
