# Free-Threading Safety Audit: jit_codegen_onednn__part1

## Files Reviewed
- `torch/csrc/jit/codegen/onednn/LlgaTensorImpl.cpp`
- `torch/csrc/jit/codegen/onednn/LlgaTensorImpl.h`
- `torch/csrc/jit/codegen/onednn/decompose_silu.cpp`
- `torch/csrc/jit/codegen/onednn/decompose_silu.h`
- `torch/csrc/jit/codegen/onednn/defer_size_check.cpp`
- `torch/csrc/jit/codegen/onednn/defer_size_check.h`
- `torch/csrc/jit/codegen/onednn/graph_fuser.cpp`
- `torch/csrc/jit/codegen/onednn/graph_fuser.h`
- `torch/csrc/jit/codegen/onednn/graph_helper.cpp`
- `torch/csrc/jit/codegen/onednn/graph_helper.h`
- `torch/csrc/jit/codegen/onednn/graph_rewriter.cpp`
- `torch/csrc/jit/codegen/onednn/guard_shape.cpp`
- `torch/csrc/jit/codegen/onednn/guard_shape.h`
- `torch/csrc/jit/codegen/onednn/interface.cpp`
- `torch/csrc/jit/codegen/onednn/interface.h`

## Overview

This group implements the oneDNN Graph (LLGA) fusion pass for TorchScript. It handles graph analysis, partitioning, subgraph creation, layout propagation, and guard insertion. The interface includes operator registration and fusion graph execution. No direct Python C API usage.

## Issues

### Issue 1: `onednn_enabled` static atomic and `RegisterLlgaFuseGraph` state

**Location:** `interface.h`, lines 9-13

```cpp
static std::atomic<bool> onednn_enabled{false};

static std::atomic<bool>& getLlgaEnabled() {
  return onednn_enabled;
}
```

These are declared `static` in a header file, which means each translation unit that includes this header gets its own copy. This is a correctness bug (not specifically a threading bug): different translation units may have different views of the "enabled" state. The `std::atomic` is itself thread-safe, but the per-TU copies defeat the purpose.

**Risk:** Medium -- this is an existing bug that manifests regardless of free-threading. The `static` in header means each TU has its own `onednn_enabled`.

**Recommended Fix:** Move the `std::atomic<bool>` into a `.cpp` file and expose via a function returning a reference, or use an `inline` variable (C++17).

### Issue 2: `RegisterLlgaFuseGraph` static methods with non-atomic `isRegistered` and `passID`

**Location:** `interface.h`, lines 19-56

`RegisterLlgaFuseGraph` inherits from `PassManager<RegisterLlgaFuseGraph>` which presumably has its own static state. The `setEnabled`, `registerPass`, and `clearPass` methods modify global pass manager state. Without seeing the `PassManager` base class implementation, this could have race conditions if `setEnabled` is called from multiple threads concurrently.

**Risk:** Medium -- if `setEnabled` can be called from multiple threads (e.g., from Python without GIL), the pass registration/clearing is not atomic as a unit.

### Issue 3: `static RegisterOperators` for oneDNN ops

**Location:** `interface.cpp`, lines 104-109, 169-174

```cpp
static RegisterOperators oneDNNFusionGroupOp({...});
static RegisterOperators oneDNNGuardOp({...});
```

These are static-initialization-time registrations. Thread-safe since they run before user threads.

**Risk:** None.

### Issue 4: `fuseGraph` uses local state only

**Location:** `interface.cpp`, lines 20-91

The `fuseGraph` function operates on a `shared_ptr<Graph>` passed by reference. All mutation is to this graph. The function uses `static std::unordered_set<Symbol> supportedOps` inside the lambda, which is read-only after initialization (function-local static, C++11 safe).

**Risk:** None, provided graphs are not shared across threads (which is the normal JIT contract).

### Issue 5: Graph helper and rewriter operate on per-graph state

**Location:** `graph_helper.cpp`, `graph_rewriter.cpp`, `graph_fuser.cpp`

`LlgaGraphHelper` creates a `dnnl::graph::graph` and an `AliasDb` per instance. `GraphRewriter` operates on a specific block. All state is local to the helper/rewriter instance.

**Risk:** None.

## Summary

The main concern is the `static std::atomic<bool> onednn_enabled` declared in a header file (Issue 1), which creates per-TU copies, making the enable/disable flag unreliable. This is an existing correctness bug independent of free-threading. The `RegisterLlgaFuseGraph::setEnabled` method (Issue 2) should be examined with the `PassManager` base class to ensure thread safety of pass registration. The rest of the code operates on local or per-instance state with no Python C API usage.
