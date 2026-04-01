# Request: Probit Estimators R Package + Monte Carlo Simulation

```yaml
RequestID: REQ-20260331-223247
Timestamp: 2026-03-31T22:32:47
TargetRepo: statsclaw/example-probit
TargetRepoLocal: .repos/example-probit
WorkspaceRepo: TianzhuQin/workspace
WorkspaceRepoLocal: .repos/workspace
Workflow: 11 (Simulation + Code + Ship)
Profile: r-package
```

## Scope

Build a complete R package (`exampleProbit`) with three probit estimation methods implemented in C++ via Rcpp/RcppArmadillo:

1. **MLE via Newton-Raphson** — `probit_mle()`: log-likelihood, gradient, Hessian; iterative optimization
2. **Bayesian Gibbs Sampler (Albert-Chib)** — `probit_gibbs()`: data augmentation with truncated normal latent variables
3. **Metropolis-Hastings MCMC** — `probit_mh()`: random-walk proposal with log-posterior evaluation

Plus shared utilities (`safe_log_Phi`, `safe_Phi`, `rtruncnorm`) and a Monte Carlo simulation study.

## Uploaded Files

- `/Users/tianzhuqin/Documents/statsclaw/probit_spec.pdf` — Full mathematical specification

## Acceptance Criteria

- C1: MLE coefficients match `glm(..., family=binomial(link="probit"))` within 0.05
- C2: Gibbs and MH posterior means within 0.1 of MLE for N >= 500
- C3: MH acceptance rate between 20% and 50%
- C4: 95% CI coverage between 90% and 99% for all methods
- C5: MLE at least 5x faster than Gibbs (C++ vs C++)
- C6: R CMD check passes with 0 errors, 0 warnings
- C7: Package installs via R CMD INSTALL and all functions callable from R

## Monte Carlo Design

- DGP: Pr(y=1|x1) = Phi(-1 + 0.5*x1), x1 ~ N(0,1)
- True parameters: beta0 = (-1, 0.5)'
- Sample sizes: N in {200, 500, 1000, 5000}
- Replications: R = 500 per scenario
- Metrics: Bias, RMSE, Coverage, Time

## Ship

Push to `statsclaw/example-probit` main branch and create PR if needed.
