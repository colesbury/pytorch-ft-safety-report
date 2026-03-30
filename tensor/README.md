# Tensor Free-Threading Issues

Issues from broad audit of `torch/csrc/tensor/`. These have NOT been deep-audited — some may be false positives (config flags, write-once globals).

## Medium Severity

| File | Issue |
|------|-------|
| `` | Mutable Static `default_backend` Without Synchronization |

*1 issues from broad audit. Deep audit needed to confirm.*
