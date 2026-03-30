# Jit/Mobile Free-Threading Issues

Issues from broad audit of `torch/csrc/jit/mobile/`. These have NOT been deep-audited — some may be false positives (config flags, write-once globals).

## Medium Severity

| File | Issue |
|------|-------|
| `jit/mobile/function.cpp` | Static mutable upgrader function holder (function.cpp) |
| `jit/mobile/prim_ops_registery.cpp` | Static mutable prim ops function table (prim_ops_registery.cpp) |
| `jit/mobile/code.h` | `Code::initialized` flag (code.h, function.cpp) |
| `jit/mobile/upgrader_mobile.cpp` | Static upgrader bytecode list in upgrader_mobile.cpp |

*4 issues from broad audit. Deep audit needed to confirm.*
