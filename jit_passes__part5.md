# Free-Threading Safety Audit: jit_passes__part5

**Overall Risk: LOW**

## Files Audited

- `jit/passes/frozen_graph_optimizations.h`
- `jit/passes/frozen_linear_folding.cpp`
- `jit/passes/frozen_linear_folding.h`
- `jit/passes/frozen_linear_transpose.cpp`
- `jit/passes/frozen_linear_transpose.h`
- `jit/passes/frozen_ops_to_mkldnn.cpp`
- `jit/passes/frozen_ops_to_mkldnn.h`
- `jit/passes/fuse_linear.cpp`
- `jit/passes/fuse_linear.h`
- `jit/passes/fuse_relu.cpp`
- `jit/passes/fuse_relu.h`
- `jit/passes/graph_fuser.cpp`
- `jit/passes/graph_fuser.h`
- `jit/passes/graph_rewrite_helper.cpp`
- `jit/passes/graph_rewrite_helper.h`

## Detailed Analysis

### frozen_graph_optimizations.h, frozen_linear_folding.cpp/.h, frozen_linear_transpose.cpp/.h

Pure graph transformation passes. All functions take a `shared_ptr<Graph>&` and operate on it locally. No static/global mutable state. No Python C API calls. No threading concerns.

**Verdict: SAFE** -- Pure graph transformations with no shared mutable state.

### frozen_ops_to_mkldnn.cpp/.h

This file has several notable patterns:

1. **Static `RegisterOperators` globals** (lines 413-606): `MKLDNNHardSwishOpReg`, `BroadOpReg`, `MKLDNNLayerNormOpReg`, `MKLDNNConstantOp`, `reg_fut_ops`. These are static initializer registrations into the global operator registry. They are constructed at load time and are themselves immutable after initialization. The thread-safety of registration depends on the operator registry infrastructure, not on this file. This is a pre-existing pattern throughout the JIT codebase and the registry itself handles synchronization.

2. **`static` local in `ConvertFrozenOpsToMKLDNN`** (line 1146): `static std::unordered_set<Symbol> mkldnn_ops` inside a lambda. This is a `const` set (never modified after initialization), initialized via C++11 static local initialization which is thread-safe.

3. All graph transformation functions (`InplaceMKLDNNSubgraph`, `ComputeSubgraphInMKLDNN`, `MKLDNNSubgraphSlicer`, `ConvertFrozenOpsToMKLDNN`) operate on a passed-in graph and use only local state.

**Verdict: SAFE** -- Static registrations are immutable after init. Graph passes use local state only.

### fuse_linear.cpp/.h

Pure graph rewriting pass using `SubgraphRewriter`. All state is local to each function call. No static mutable state, no Python C API.

**Verdict: SAFE**

### fuse_relu.cpp/.h

Pure graph rewriting pass using `SubgraphRewriter`. All state is local. No static mutable state, no Python C API.

**Verdict: SAFE**

### graph_fuser.cpp/.h

1. **`static bool cpu_fuser_enabled_legacy`** (line 1240): A file-scope mutable boolean, read by `canFuseOnCPULegacy()` and written by `overrideCanFuseOnCPULegacy()`. Under free-threading, concurrent read/write to this non-atomic boolean is a data race.

   - **Risk: LOW** -- This is a configuration flag typically set once before use. The race window is small and the consequence is merely reading a stale boolean. However, for correctness it should be `std::atomic<bool>`.

2. **`static OperatorSet simple_mappable`** (line 32): Declared inside `isSimpleMap()` as a function-local static. This is initialized once (thread-safe via C++11) and never modified afterward. Safe.

3. The `GraphFuser` struct and all fusion logic operate on passed-in `Block*`/`Graph*` with local state only.

**Verdict: LOW RISK** -- The `cpu_fuser_enabled_legacy` static bool should be `std::atomic<bool>`, but it is a configuration flag with minimal race impact.

### graph_rewrite_helper.cpp/.h

Pure utility functions (`getFuncName`, `getValue`, `getIValue`, `replaceConvolutionWithAtenConv`, `isClampFusable`). All operate on passed-in parameters with no static mutable state. No Python C API.

**Verdict: SAFE**

## Issues Found

| # | Severity | File | Line | Description |
|---|----------|------|------|-------------|
| 1 | Low | `graph_fuser.cpp` | 1240 | `static bool cpu_fuser_enabled_legacy` is a non-atomic mutable global read/written by `canFuseOnCPULegacy()`/`overrideCanFuseOnCPULegacy()`. Should be `std::atomic<bool>`. |

## Summary

This group of files is predominantly safe for free-threading. The files consist almost entirely of pure C++ graph transformation passes that operate on passed-in graph objects with local-only state. The one concern is a non-atomic static boolean configuration flag in `graph_fuser.cpp`, which is a minor issue since it is a configuration toggle typically set once early in program execution.
