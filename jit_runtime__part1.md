# Free-Threading Safety Audit: jit_runtime__part1

## Files Reviewed
- `jit/runtime/argument_spec.cpp`
- `jit/runtime/argument_spec.h`
- `jit/runtime/autodiff.cpp`
- `jit/runtime/autodiff.h`
- `jit/runtime/calculate_necessary_args.h`
- `jit/runtime/custom_operator.h`
- `jit/runtime/decomposition_registry.cpp`
- `jit/runtime/decomposition_registry.h`
- `jit/runtime/decomposition_registry_util.cpp`
- `jit/runtime/decomposition_registry_util.h`
- `jit/runtime/exception_message.h`
- `jit/runtime/graph_executor.cpp`
- `jit/runtime/graph_executor.h`
- `jit/runtime/graph_executor_impl.h`
- `jit/runtime/graph_iterator.h`

## Summary

This group contains the core JIT graph executor and related infrastructure. The code is predominantly C++-only and does not interact with the Python C API directly. The graph executor already uses mutexes (`compile_mutex` in `GraphExecutorImplBase`) to protect its compilation caches, and atomic variables for global mode flags. However, there are several mutable global/static state patterns that pose thread-safety risks under free-threading.

## Issues

### Issue 1
- **Category**: Static/global mutable state
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/decomposition_registry.cpp`
- **Lines**: 19-30
- **Description**: Several global mutable maps (`schema_to_decomposition`, `user_registered_funcs`, `schema_to_function`) and a shared `compilation_unit` are defined at namespace scope. The `loadDecompositionFunctions()` function uses the mutex `lock` to guard lazy initialization, but `GetDecomposition()` and `GetDecompositionFunction()` call `loadDecompositionFunctions()` and then read from these maps without holding the lock. After the initial load, these reads are racing with `RegisterDecomposition()` which writes to the maps while holding the lock. The lock only protects the first load, not subsequent concurrent reads/writes.
- **Fix**: Either hold the mutex for the entire duration of `GetDecomposition` / `GetDecompositionFunction`, or use a reader-writer lock. Alternatively, make `RegisterDecomposition` also use the same lock consistently and have `GetDecomposition`/`GetDecompositionFunction` acquire the lock when reading.

### Issue 2
- **Category**: Lazy initialization pattern
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/decomposition_registry.cpp`
- **Lines**: 48-65
- **Description**: `loadDecompositionFunctions()` uses a check-then-act pattern (`if (!schema_to_decomposition.empty()) return;`) inside the lock, which is correct for the double-checked locking pattern itself. However, the callers (`GetDecomposition`, `GetDecompositionFunction`) read from `schema_to_decomposition` and `schema_to_function` without holding the lock after calling `loadDecompositionFunctions()`. Under free-threading, a concurrent call to `RegisterDecomposition()` could modify these maps while another thread reads them.
- **Fix**: Extend the lock scope to cover the reads in `GetDecomposition` and `GetDecompositionFunction`, or use a separate read lock.

### Issue 3
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/graph_executor.cpp`
- **Lines**: 84, 95-96, 104
- **Description**: `autodiff_subgraph_inlining` is `thread_local` (safe). `fusion_group_inlining` is `std::atomic<bool>` (safe). `last_executed_optimized_graph` is `thread_local` (safe). These are properly handled.
- **Fix**: No fix needed.

### Issue 4
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/autodiff.cpp`
- **Lines**: 28-32, 41-44
- **Description**: `needTrimGrad` and `isDifferentiable` use `static OperatorSet` variables. These are initialized once and then only read, so they are safe as long as `OperatorSet` construction is itself thread-safe (it is constructed at first call under the function-local static guarantee of C++11).
- **Fix**: No fix needed; C++11 guarantees thread-safe initialization of function-local statics.

### Issue 5
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/graph_executor.cpp`
- **Lines**: 544-549
- **Description**: `static RegisterOperators reg_graph_executor_ops(...)` registers operators during static initialization. This is a one-time operation before `main()` and does not recur, so it is safe.
- **Fix**: No fix needed.

### Issue 6
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/graph_executor.cpp`
- **Lines**: 863-868
- **Description**: `IsNewExecutorEnabled()` uses a function-local `static const auto disable_new_executor` which is safely initialized once (C++11 guarantee). No issue.
- **Fix**: No fix needed.
