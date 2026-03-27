# Free-Threading Safety Audit: jit_passes_utils

## Overall Risk: LOW

## Files Audited
- `jit/passes/utils/check_alias_annotation.cpp`
- `jit/passes/utils/check_alias_annotation.h`
- `jit/passes/utils/memory_dag.cpp`
- `jit/passes/utils/memory_dag.h`
- `jit/passes/utils/op_registry.cpp`
- `jit/passes/utils/op_registry.h`
- `jit/passes/utils/optimization_utils.cpp`
- `jit/passes/utils/optimization_utils.h`
- `jit/passes/utils/subgraph_utils.cpp`
- `jit/passes/utils/subgraph_utils.h`

## Summary

All files in this group are utility libraries for JIT pass infrastructure. They provide data structures (memory DAG, alias checking) and helper functions (subgraph manipulation, operator registry queries) that operate on passed-in parameters. There is no static/global mutable state, no Python C API usage, and no lazy initialization patterns.

## Detailed Analysis

### No Issues Found

1. **check_alias_annotation.cpp/h**: Functions that verify alias annotations by running ops and checking aliasing properties. All state is local.

2. **memory_dag.cpp/h**: `MemoryDAG` and `MemoryDAGBuilder` classes for tracking aliasing relationships. All instances are created locally by `AliasDb`.

3. **op_registry.cpp/h**: Provides `OperatorSet` lookups for shape analysis. No static mutable state observed in grep results.

4. **optimization_utils.cpp/h**: Utility functions for graph optimization.

5. **subgraph_utils.cpp/h**: Functions for creating, merging, and manipulating subgraphs. Contains local static helper functions (`collectNestedUses`, `closedOverValues`, `truncateStrWithHash`) that operate on passed-in parameters.

## Recommendations

None. These files are safe for free-threading.
