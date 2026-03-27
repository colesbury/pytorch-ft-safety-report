# Free-Threading Safety Audit: jit_runtime_interpreter

## Files Reviewed
- `jit/runtime/interpreter/can_emit_inline.h`
- `jit/runtime/interpreter/code_impl.h`
- `jit/runtime/interpreter/frame.cpp`
- `jit/runtime/interpreter/frame.h`
- `jit/runtime/interpreter/preprocess_graph.cpp`
- `jit/runtime/interpreter/preprocess_graph.h`

## Summary

This group contains the JIT interpreter's internal implementation: the `CodeImpl` structure, frame management, graph preprocessing, and inline emission analysis. The code is predominantly internal C++ with no direct Python C API interactions. The main state management is per-`CodeImpl` or per-`Frame` instance, not global. The only global state is a frame ID counter that uses `std::atomic`.

## Issues

### Issue 1
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: High
- **File**: `jit/runtime/interpreter/frame.cpp`
- **Lines**: 6-9
- **Description**: `Frame::genId()` uses a `static std::atomic<size_t> numFrames{0}` with `fetch_add` and `memory_order_relaxed`. This is a properly atomic counter used to generate unique frame IDs. Thread-safe.
- **Fix**: No fix needed.

### Issue 2
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/interpreter/code_impl.h`
- **Lines**: 68-145
- **Description**: `CodeImpl` contains substantial mutable state (instruction tables, operator tables, function tables, etc.), but instances are created during compilation and then shared via `std::shared_ptr`. Once compiled, the `CodeImpl` is effectively immutable -- it is read by `InterpreterState` during execution. The `grad_executors_` and `forward_executors_` optional vectors are lazily populated but accessed through the `Code` wrapper. In the current architecture, compilation is protected by the `compile_mutex` in `GraphExecutorImplBase`, so the `CodeImpl` is fully constructed before being shared. No data race.
- **Fix**: No fix needed.

### Issue 3
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/interpreter/can_emit_inline.h`
- **Lines**: 36-104
- **Description**: `CanEmitInline` stores results in a `std::unordered_map<Node*, bool> can_emit_inline_`. This is created fresh for each graph compilation and is not shared across threads. Safe.
- **Fix**: No fix needed.

### Issue 4
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/interpreter/preprocess_graph.cpp`
- **Lines**: 1-100
- **Description**: `PreprocessGraph` modifies a copy of the graph during construction. The resulting `graph` and `can_emit_inline` map are stored in the struct and used by `CodeImpl`. Since preprocessing happens during compilation (under the compile mutex), this is safe.
- **Fix**: No fix needed.
