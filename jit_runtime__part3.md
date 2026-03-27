# Free-Threading Safety Audit: jit_runtime__part3

## Files Reviewed
- `jit/runtime/profiling_graph_executor_impl.cpp`
- `jit/runtime/profiling_graph_executor_impl.h`
- `jit/runtime/profiling_record.cpp`
- `jit/runtime/profiling_record.h`
- `jit/runtime/register_c10_ops.cpp`
- `jit/runtime/register_cuda_ops.cpp`
- `jit/runtime/register_distributed_ops.cpp`
- `jit/runtime/register_ops_utils.cpp`
- `jit/runtime/register_ops_utils.h`
- `jit/runtime/register_prim_ops.cpp`
- `jit/runtime/register_prim_ops_fulljit.cpp`
- `jit/runtime/register_special_ops.cpp`
- `jit/runtime/script_profile.cpp`
- `jit/runtime/script_profile.h`
- `jit/runtime/serialized_shape_function_registry.cpp`

## Summary

This group contains the profiling graph executor, profiling record infrastructure, operator registration files, script profiling, and serialized shape function registry. The profiling executor properly uses the inherited `compile_mutex`. The `ProfileRegistry` and `ProfilesRegistry` use their own mutexes. The global atomic variables for executor/profiling modes are safe. The main concerns are a global `FusionStrategy` protected by a mutex but with a nullable optional pattern, and the `py::gil_scoped_acquire` usage in the distributed ops file.

## Issues

### Issue 1
- **Category**: Static/global mutable state
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/profiling_graph_executor_impl.cpp`
- **Lines**: 100, 119-137
- **Description**: `fusion_strategy_lock` protects `fusion_strategy` (a `std::optional<FusionStrategy>`). The `getFusionStrategy()` and `setFusionStrategy()` functions properly acquire the lock before accessing `fusion_strategy`. This is correctly synchronized.
- **Fix**: No fix needed.

### Issue 2
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: High
- **File**: `jit/runtime/profiling_graph_executor_impl.cpp`
- **Lines**: 93-98, 139, 141-156
- **Description**: `executor_mode`, `profiling_mode`, and `num_profiled_runs` are all `std::atomic` types. `getProfilingMode()`, `getExecutorMode()`, and `getNumProfiledRuns()` return references to them. Returning a reference to an atomic is safe for load/store operations -- callers can safely do `getProfilingMode().load()` or assign to the atomic. The function-local static in `getNumProfiledRuns()` for initialization is also safely guarded by C++11 function-local static initialization.
- **Fix**: No fix needed.

### Issue 3
- **Category**: Python C API / GIL interaction
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/register_distributed_ops.cpp`
- **Lines**: 56-58
- **Description**: `py::gil_scoped_acquire acquire;` is used to acquire the GIL before calling `get_python_cu()`. Under free-threading (Python 3.14t/nogil), `py::gil_scoped_acquire` is a no-op or acquires a critical section, but `get_python_cu()` may access Python state that needs explicit protection. If the compilation unit returned is accessed after the GIL is released, this could be a race.
- **Fix**: Ensure that the returned `cuPtr` from `get_python_cu()` is a `shared_ptr` that keeps the compilation unit alive and that the function schema accessed from it is immutable. Under free-threading, verify that `get_python_cu()` uses appropriate locking.

### Issue 4
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: High
- **File**: `jit/runtime/register_c10_ops.cpp`
- **Lines**: 37-52
- **Description**: `Registerer` is a function-local static that sets up a listener on the dispatcher. This is initialized exactly once (C++11 guarantee) and the listener's `onOperatorRegistered`/`onOperatorDeregistered` callbacks call into the JIT operator registry which has its own mutex. Safe.
- **Fix**: No fix needed.

### Issue 5
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/script_profile.cpp`
- **Lines**: 15-52
- **Description**: `ProfilesRegistry` uses a mutex to protect `enabledProfiles_` and an `std::atomic<bool>` for fast-path checking (`empty()`). The atomic uses `memory_order_relaxed`, which means a thread may observe a stale value and skip profiling or unnecessarily enter the locked path. This is acceptable for a profiling system where occasional missed or extra profiling is harmless.
- **Fix**: No fix needed; the relaxed ordering is a deliberate performance optimization.

### Issue 6
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/script_profile.cpp`
- **Lines**: 54-104
- **Description**: `initBindings()` registers TorchBind classes during static initialization via the `torchBindInitializer` variable. This is a one-time operation. Safe.
- **Fix**: No fix needed.

### Issue 7
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/profiling_record.cpp`
- **Lines**: 16-52
- **Description**: `ProfileRegistry` uses a function-local static singleton with its own mutex. `registerProfileNode()` and `shouldProfileNode()` both acquire the lock. This is properly synchronized.
- **Fix**: No fix needed.
