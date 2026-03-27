# Free-Threading Safety Audit: jit_passes__part8

**Overall Risk: MEDIUM**

## Files Audited

- `jit/passes/metal_rewrite.cpp`
- `jit/passes/metal_rewrite.h`
- `jit/passes/mkldnn_rewrite.cpp`
- `jit/passes/mkldnn_rewrite.h`
- `jit/passes/mobile_optimizer_type.h`
- `jit/passes/normalize_ops.cpp`
- `jit/passes/normalize_ops.h`
- `jit/passes/onednn_graph_fuser.h`
- `jit/passes/onnx.cpp`
- `jit/passes/onnx.h`
- `jit/passes/pass_manager.cpp`
- `jit/passes/pass_manager.h`
- `jit/passes/peephole.cpp`
- `jit/passes/peephole.h`
- `jit/passes/peephole_alias_sensitive.cpp`

## Detailed Analysis

### metal_rewrite.cpp/.h

Pure graph rewriting pass using `SubgraphRewriter`. Functions like `insertPrePackedLinearOp`, `insertPrePackedConv2dOp`, `fuseReluWithPackedOps`, `fuseHardtanhWithPackedOps` operate on passed-in graphs/modules with local state. `metalOptimizeForMobile` clones the module before modifying it.

**Verdict: SAFE** -- All state is local.

### mkldnn_rewrite.cpp/.h

Graph transformation pass that inserts MKLDNN pre-packed convolution operations and fuses with element-wise ops. All functions operate on passed-in graphs with local state.

The `mkldnn::fusion_rewrite_map` in `mkldnn_rewrite.h` (line 20) is `const static` -- immutable after initialization.

**Verdict: SAFE** -- All state is local.

### mobile_optimizer_type.h

An enum definition only. No mutable state.

**Verdict: SAFE**

### normalize_ops.cpp/.h

`getOperatorAliasMap()` returns a reference to a `static const std::unordered_map<Symbol, Symbol>` (line 81). This is initialized once (thread-safe via C++11 static local) and is never modified. Safe.

`NormalizeOps` operates on a passed-in graph with local state.

**Verdict: SAFE**

### onednn_graph_fuser.h

This file has notable thread-safety concerns:

1. **`static std::atomic<bool> onednn_enabled{true}`** (line 12): Declared at namespace scope but with `static` linkage, meaning each translation unit that includes this header gets its own copy. This is likely a bug (values could be out of sync across TUs), but since it is atomic, there is no data race within a single TU.

2. **`RegisterLlgaFuseGraph`** inherits from `PassManager<RegisterLlgaFuseGraph>`. The `setEnabled`/`isEnabled` methods use the atomic and call `registerPass`/`clearPass` which modify the global pass registry (see `pass_manager.cpp` below).

**Verdict: LOW RISK** -- The atomic is per-TU (a design issue but not a free-threading race). The pass registration calls into `pass_manager.cpp` which has its own issues (see below).

### onnx.cpp/.h

1. **Python C API usage**: `onnx.cpp` makes extensive use of the Python C API through `pybind11`:
   - `py::module::import(...)` calls in `NodeToONNX` (lines 260-265)
   - `py::dict env`, `py::set values_in_env` passed around and mutated
   - `py::cast(...)` calls throughout
   - `py::object` attribute access (`onnx_globals.attr(...)`, etc.)

   All Python C API usage in this file occurs within functions that are called from Python (ONNX export path), so the GIL would be held in the traditional model. Under free-threading, these Python object manipulations would need critical sections if called from multiple threads. However, ONNX export is typically a single-threaded operation and these functions are called from Python code that would naturally serialize.

2. **`ConstantValueMap::ClearMaps()` / `SetAllGraphInputsStatic()`**: These call into static methods on `ConstantValueMap` which likely manage global state. Thread-safety depends on the `ConstantValueMap` implementation.

**Verdict: LOW RISK** -- Python C API use is extensive but occurs in code paths that are naturally single-threaded (ONNX export). Concurrent ONNX exports would be unsafe without external synchronization, but this is unlikely in practice.

### pass_manager.cpp/.h

This file manages global pass registries and has significant thread-safety concerns:

1. **`static GraphPassNameType graphPassID = 1`** (line 6): A file-scope mutable counter incremented by `registerPostPass` and `registerPrePass`. Not atomic, not protected by any lock. Concurrent registration from multiple threads would race on this counter.

2. **`getCustomPostPasses()` and `getCustomPrePasses()`** (lines 8-16): Return references to static `std::vector<GraphPassEntry>`. These vectors are mutated by `registerPostPass`, `registerPrePass`, `clearPostPass`, `clearPrePass`, `clearAllPostPasses`, `clearAllPrePasses`. They are also iterated over by the graph executor when running passes. Without synchronization, concurrent registration/clearing and iteration is a data race.

3. **`PassManager` template** (pass_manager.h): The `isRegistered()` and `passID()` methods use static local variables with a "flip_bit" pattern to toggle state. This is not thread-safe -- concurrent calls to `registerPass`/`clearPass` on the same `PassManager<T>` specialization would race on these statics.

**Verdict: MEDIUM RISK** -- The pass manager's global mutable state (pass ID counter, pass vectors, PassManager template statics) is not thread-safe. Registration typically happens during initialization or is gated by other mechanisms, but nothing prevents concurrent access. Under free-threading, concurrent `registerPass`/`clearPass` calls or concurrent pass registration + graph execution would be data races.

### peephole.cpp/.h

The `PeepholeOptimizeImpl` struct holds instance state (`graph_`, `shape_peepholes_`). The `FuseAddMM` function operates on a passed-in block with local state. No static mutable state.

**Verdict: SAFE**

### peephole_alias_sensitive.cpp

The `PeepholeOptimizeAliasSensitiveImpl` struct holds instance state (`graph_`, `aliasDb_`, `shape_peepholes_`, `stale_alias_values_`). All state is local.

**Verdict: SAFE**

## Issues Found

| # | Severity | File | Line | Description |
|---|----------|------|------|-------------|
| 1 | Medium | `pass_manager.cpp` | 6 | `static GraphPassNameType graphPassID` is a non-atomic mutable global counter used by `registerPostPass`/`registerPrePass`. Concurrent registration would race. |
| 2 | Medium | `pass_manager.cpp` | 8-16 | `getCustomPostPasses()`/`getCustomPrePasses()` return mutable static vectors that are mutated by register/clear functions and iterated by the graph executor. No synchronization. |
| 3 | Medium | `pass_manager.h` | 84-104 | `PassManager<T>::isRegistered()` and `PassManager<T>::passID()` use static local variables modified via a "flip_bit" pattern. Concurrent calls to `registerPass`/`clearPass` on the same specialization race on these statics. |
| 4 | Low | `onednn_graph_fuser.h` | 12 | `static std::atomic<bool> onednn_enabled` has `static` linkage in a header, meaning each TU gets a separate copy. This is a design bug (inconsistent state across TUs) but not a data race since it is atomic. |
| 5 | Low | `onnx.cpp` | 260-265 | Extensive Python C API usage (pybind11) without explicit critical sections. Safe only because ONNX export is naturally single-threaded. |

## Summary

Most files in this group are pure C++ graph transformation passes with no shared mutable state (metal_rewrite, mkldnn_rewrite, normalize_ops, peephole, peephole_alias_sensitive). The primary concern is `pass_manager.cpp/.h`, which manages global mutable state (pass registries and a pass ID counter) without any synchronization. Under free-threading, concurrent pass registration/deregistration or concurrent registration with pass execution would be data races. The `onnx.cpp` file uses the Python C API extensively but in a naturally single-threaded context. The `onednn_graph_fuser.h` has a header-static atomic that works per-TU but may lead to inconsistent state across translation units.
