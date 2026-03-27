# Free-Threading Safety Audit: jit_runtime_static__part1

## Files Reviewed
- `jit/runtime/static/ProcessedNodeInputs.cpp`
- `jit/runtime/static/ProcessedNodeInputs.h`
- `jit/runtime/static/fusion.cpp`
- `jit/runtime/static/fusion.h`
- `jit/runtime/static/generated_ops.cpp`
- `jit/runtime/static/impl.cpp`
- `jit/runtime/static/impl.h`
- `jit/runtime/static/init.cpp`
- `jit/runtime/static/init.h`
- `jit/runtime/static/memory_planner.cpp`
- `jit/runtime/static/memory_planner.h`
- `jit/runtime/static/native_ops.cpp`
- `jit/runtime/static/ops.cpp`
- `jit/runtime/static/ops.h`
- `jit/runtime/static/passes.cpp`

## Summary

This group contains the Static Runtime implementation, which is an optimized inference runtime for TorchScript models. The code is largely C++ with minimal Python interaction (only in `init.cpp` for Python bindings). Static Runtime is designed for single-threaded inference per `StaticRuntime` instance, but `StaticModule` can be shared. The main thread-safety concerns relate to global operator registries and the Python binding code in `init.cpp`.

## Issues

### Issue 1
- **Category**: Python C API interaction
- **Severity**: Medium
- **Confidence**: Medium
- **File**: `jit/runtime/static/init.cpp`
- **Lines**: 14-100
- **Description**: `initStaticModuleBindings()` uses pybind11 to define Python classes and methods. The lambdas in `__call__` and other methods use `torch::jit::toIValue()` and `toPyObject()` which interact with the Python C API. Under free-threading, these Python object conversions need critical section protection if called concurrently. However, since these are called from Python through pybind11, they will naturally be called with the GIL held (or equivalent protection in free-threaded mode), so pybind11's automatic GIL management should handle this.
- **Fix**: Verify that pybind11's GIL management works correctly with Python 3.14t. No immediate code changes needed if using a free-threading-aware pybind11 version.

### Issue 2
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/static/impl.cpp`
- **Lines**: 47-50
- **Description**: `C10_DEFINE_bool(static_runtime_disable_debug_memory_overlap_check, ...)` defines a global flag. This is accessed via the gflags mechanism and is effectively set once at startup. Safe for read-only access during runtime.
- **Fix**: No fix needed.

### Issue 3
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/static/impl.h`
- **Lines**: 52-61
- **Description**: `borrowsOutputs()` uses a `static const std::array` which is initialized once and only read. C++11 guarantees thread-safe initialization. Safe.
- **Fix**: No fix needed.

### Issue 4
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/static/ops.h`
- **Lines**: 33-52
- **Description**: `TORCH_DECLARE_REGISTRY(SROperatorRegistry, ...)` and `TORCH_DECLARE_REGISTRY(SRNativeOperatorRegistry, ...)` declare global registries. Registration happens during static initialization (via `C10_REGISTER_CLASS` macros). The registries are read during `StaticModule` construction but not modified after startup. Under typical usage, this is safe. However, if dynamic registration were to occur at runtime concurrently with lookups, this could race. The C10 registry implementation typically uses a mutex internally.
- **Fix**: No fix needed for typical usage patterns.
