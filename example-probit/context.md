# Repository Context

```yaml
RepoName: "example-probit"
RepoURL: "https://github.com/statsclaw/example-probit"
RepoCheckout: "/Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/example-probit"
ActiveRun: "probit-20260401-103705"
DefaultWorkflow: agent-teams
DefaultProfile: "r-package"
DefaultBranch: "main"
Language: "R / C++ (Rcpp)"
Version: "0.1.0"
CredentialStatus: "PASS"
CredentialMethod: "credential-helper"
CredentialVerifiedAt: "2026-04-01 10:37"
```

## Request Defaults

- Default acceptance criteria: R CMD check with 0 errors, 0 warnings
- Default write surface: full package (new repo)
- Default validation level: full (build + check + test)

## Key Functions

- `probit_mle()` — MLE via Newton-Raphson (C++)
- `probit_gibbs()` — Bayesian Gibbs sampler, Albert-Chib (C++)
- `probit_mh()` — Random-walk Metropolis-Hastings (C++)
- `run_simulation()` — Monte Carlo simulation study (R)

## Constraints

- All estimators implemented in C++ via Rcpp/RcppArmadillo
- Shared utilities in utils.cpp (safe_log_Phi, safe_Phi, rtruncnorm)
- Numerical stability: use R::pnorm for log-space CDF computation
- Pipeline isolation: code specs, test specs, and simulation specs are separate

## Known Issues

_New repository — no known issues._

## Session Notes

_New repo build from PDF specification._
