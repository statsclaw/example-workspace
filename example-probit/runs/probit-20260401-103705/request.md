# Request

```yaml
Request ID: probit-20260401-103705
Target Repo: statsclaw/example-probit
Workspace Repo: TianzhuQin/workspace (PASS)
Workflow: 11 (Simulation + Code)
Profile: r-package
Requested By: user
Timestamp: 2026-04-01 10:37
```

## Scope

Build an R package (`exampleProbit`) from a PDF specification containing three probit estimation methods implemented in C++ via Rcpp/RcppArmadillo:

1. **MLE via Newton-Raphson** — log-likelihood, gradient, Hessian; iterate to convergence
2. **Bayesian Gibbs Sampler (Albert-Chib)** — data augmentation with truncated normal latent variables
3. **Random-walk Metropolis-Hastings** — log-posterior evaluation with adaptive proposal

Then run a **Monte Carlo simulation study** comparing all three methods on:
- Bias, RMSE, 95% CI coverage, computational speed
- Sample sizes N = {200, 500, 1000, 5000}, R = 500 replications per scenario
- DGP: Pr(y=1|x1) = Phi(-1 + 0.5*x1), x1 ~ N(0,1), true beta = (-1, 0.5)'

## Acceptance Criteria

- C1: MLE matches glm(..., family=binomial(link="probit")) within 0.05
- C2: Gibbs and MH posterior means within 0.1 of MLE for N >= 500
- C3: MH acceptance rate between 20% and 50%
- C4: 95% CI coverage between 90% and 99% for all methods
- C5: MLE at least 5x faster than Gibbs (C++ vs C++)
- C6: R CMD check passes with 0 errors, 0 warnings
- C7: Package installs via R CMD INSTALL and all functions callable from R

## Uploaded Files

- `/Users/tianzhuqin/Documents/statsclaw/probit_spec.pdf` — full implementation specification

## Ship Instructions

Ship to statsclaw/example-probit (main branch). Save all workflow documents in workspace repo.
