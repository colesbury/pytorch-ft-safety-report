# Export Free-Threading Issues

Issues from broad audit of `torch/csrc/export/`. These have NOT been deep-audited — some may be false positives (config flags, write-once globals).

## Medium Severity

| File | Issue |
|------|-------|
| `` | Unprotected Global Mutable State: `upgrader_registry` |

*1 issues from broad audit. Deep audit needed to confirm.*
