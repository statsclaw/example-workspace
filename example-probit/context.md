# Repository Context

```yaml
RepoName: "example-probit"
RepoURL: "https://github.com/statsclaw/example-probit"
RepoCheckout: ".repos/example-probit"
ActiveRun: "20260331-interflex-py-224717"
DefaultWorkflow: agent-teams
DefaultProfile: "python-package"
DefaultBranch: "main"
Language: "Python"
Version: "0.1.0"
CredentialStatus: "PASS"
CredentialMethod: "gh-cli"
CredentialVerifiedAt: "2026-03-31 22:47"
```

## Request Defaults

- Default acceptance criteria: pytest passes, package installs cleanly
- Default write surface: determined by impact.md per run
- Default validation level: full (install + test)

## Key Functions

- `interflex()` — main entry point for linear interaction model estimation
- `interflex_linear()` — core linear estimator with delta/simulation/bootstrap variance

## Constraints

- Translate only `estimator="linear"` from R interflex package
- Python package using numpy, pandas, statsmodels, scipy
- Support discrete and continuous treatments
- Support method="linear" (OLS), plus logit/probit/poisson as secondary
- Support variance types: delta, simulation, bootstrap
- Support robust/cluster/homoscedastic standard errors

## Known Issues

- Empty repo — no existing code. Fresh Python package creation.

## Session Notes

- Source: xuyiqing/interflex R package (github.com/xuyiqing/interflex)
- Translating linear.R (~2249 lines) + relevant parts of interflex.R + vcluster.R
