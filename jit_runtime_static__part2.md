# Free-Threading Safety Audit: jit_runtime_static__part2

## Files Reviewed
- `jit/runtime/static/passes.h`
- `jit/runtime/static/processed_node_wrapper.h`
- `jit/runtime/static/static_method.h`
- `jit/runtime/static/te_wrapper.cpp`
- `jit/runtime/static/te_wrapper.h`

## Summary

This group contains the remaining Static Runtime files: passes, a processed node wrapper, a static method helper, and the TensorExpr (TE) wrapper for NNC-compiled kernels. The main thread-safety concern is in the NNC cache, which uses a mutex-protected global map. The rest of the code is either header-only utilities or per-instance state.

## Issues

### Issue 1
- **Category**: Static/global mutable state with lazy initialization
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/static/te_wrapper.cpp`
- **Lines**: 90-114
- **Description**: `getNNCCacheMutex()` and `getNNCCache()` provide a global mutex and a global `FastMap<NodeKind, shared_ptr<TEWrapper>>` cache, respectively. Both are function-local statics, safely initialized once. `lookupNNCCache()` and `updateNNCCache()` properly acquire the mutex before accessing the cache. However, the `create*()` functions (e.g., `createDiv`, `createRelu`, etc.) have a check-then-act pattern: they call `lookupNNCCache()` which acquires/releases the lock, then conditionally call `updateNNCCache()` which acquires/releases the lock. Between these two calls, another thread could also find the cache empty and start building the same TEWrapper. This results in redundant computation but not a data race, since `updateNNCCache()` simply overwrites the entry under the lock.
- **Fix**: No fix strictly needed -- the worst case is redundant compilation of the same NNC kernel, which is harmless. To avoid redundant work, the check and update could be combined under a single lock acquisition.

### Issue 2
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/static/te_wrapper.cpp`
- **Lines**: 234
- **Description**: `createClamp()` uses a function-local `static auto clamp_symbol = c10::Symbol::fromQualString("aten::clamp")`. `Symbol::fromQualString` returns a `Symbol` which is a simple integer-like type, and the function-local static initialization is thread-safe per C++11. Safe.
- **Fix**: No fix needed.

### Issue 3
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/static/te_wrapper.h`
- **Lines**: 11-33
- **Description**: `TEWrapper` contains per-instance state (`cg` unique_ptr to `LLVMCodeGen`). Instances are stored in the NNC cache and shared via `shared_ptr`. Once created and inserted into the cache, the `TEWrapper` is effectively immutable (only `call()` is invoked, which calls into the compiled code). The `call()` method on `LLVMCodeGen` should be safe for concurrent invocation as it only reads compiled code. Safe.
- **Fix**: No fix needed.

### Issue 4
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/static/passes.h`
- **Description**: Header file declaring pass functions. No mutable state. Safe.
- **Fix**: No fix needed.

### Issue 5
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/static/processed_node_wrapper.h`
- **Description**: Header file defining a wrapper struct. No global mutable state. Safe.
- **Fix**: No fix needed.

### Issue 6
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/static/static_method.h`
- **Description**: Header file defining a `StaticMethod` struct. Per-instance state only. Safe.
- **Fix**: No fix needed.
