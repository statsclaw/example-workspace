# Request

```yaml
RequestID: issue-1-20260331-230107
Package: example-fect
TargetRepo: statsclaw/example-fect
TargetCheckout: /Users/tianzhuqin/Documents/Github/statsclaw/statsclaw/.repos/example-fect
WorkspaceRepo: .repos/workspace
WorkspaceRepoStatus: NOT_AVAILABLE
Workflow: 5 (Single Issue Fix)
Profile: r-package
```

## Scope

Fix GitHub issue #1: "Convergence performance and robustness improvements"

Two sub-problems:
1. **Convergence speed** — EM/optimization loop converges slowly on moderate-to-large panels (N > 200, T > 30). Iteration counts higher than expected for matrix-completion approaches.
2. **Robustness of results** — Estimates sensitive to initial conditions and tuning parameter choices. Small perturbations in data/hyperparameters lead to non-trivial changes in point estimates.

## Acceptance Criteria

1. Convergence iteration counts reduced measurably on moderate-to-large panels
2. Estimates stable across different initial conditions and tuning parameters
3. R CMD check passes with no errors or warnings
4. All existing tests pass
5. No regression in accuracy of estimates
