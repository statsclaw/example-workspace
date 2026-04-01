# Handoff — example-probit

> Last updated: 2026-04-01 | Run: probit-20260401-103705

## Active Handoff Notes

- The package is ready for review and shipping. All 7 acceptance criteria pass. The ARCHITECTURE.md is in the target repo root.
- The R CMD check LaTeX-related error/warning is from missing pdflatex on the build system, not a package defect. On a system with a LaTeX distribution, these would not appear.
- The MH scale parameter (2.4) was empirically tuned for the 2-dimensional probit model used in the simulation DGP. For higher-dimensional models, users may need to adjust the scale parameter to maintain good acceptance rates.
- The simulation takes approximately 30-60 minutes to run the full grid (6000 estimator fits). The exported function run_probit_simulation() accepts reduced R for faster testing.
- The 4 NOTEs from R CMD check are standard for new submissions and do not indicate problems: "New submission", RcppArmadillo not directly imported from R code (it is used via LinkingTo), HTML tidy unavailable, and a leftover tex file from the PDF manual attempt.
