# Fx Free-Threading Issues

Issues from broad audit of `torch/csrc/fx/`. These have NOT been deep-audited — some may be false positives (config flags, write-once globals).

## High Severity

| File | Issue |
|------|-------|
| `` | Lazy-Initialized Static `PyObject*` Without Synchronization |

## Medium Severity

| File | Issue |
|------|-------|
| `` | Borrowed References from Python Containers Without Critical Section |
| `` | Direct Python Object Mutation Without Critical Section |

*3 issues from broad audit. Deep audit needed to confirm.*
