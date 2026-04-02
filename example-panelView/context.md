# Repository Context

```yaml
RepoName: "example-panelView"
RepoURL: "https://github.com/statsclaw/example-panelView.git"
RepoCheckout: "/Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/example-panelView"
ActiveRun: ""
DefaultWorkflow: agent-teams
DefaultProfile: "r-package"
DefaultBranch: "main"
Language: "R"
Version: "1.2.1"
CredentialStatus: ""
CredentialMethod: ""
CredentialVerifiedAt: ""
```

## Request Defaults

- Default acceptance criteria: all profile validation commands pass with zero errors
- Default write surface: determined by impact.md per run
- Default validation level: full (build + check + test)

## Key Functions

- `panelview()` — sole exported function, dispatches to `.pv_plot_treat()`, `.pv_plot_outcome()`, `.pv_plot_bivariate()` based on `type` parameter
- Observation matrix `I` (TT × N) — binary indicator of unit-in-period, constructed during data reshaping (panelView.R:625-995)
- `obs.missing` matrix — unified encoding with treatment status overlay

## Constraints

- Single-function package: all features route through `panelview()`
- ggplot2 (>= 3.4.0) is a hard dependency (Imports)
- igraph must go in Suggests, not Imports (user requirement)
- gridExtra, grid, dplyr (>= 1.0.0) in Imports
- Only testthat in current Suggests

## Known Issues

_No known issues yet._

## Session Notes

_No notes yet._
