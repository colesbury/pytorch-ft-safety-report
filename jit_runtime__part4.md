# Free-Threading Safety Audit: jit_runtime__part4

## Files Reviewed
- `jit/runtime/serialized_shape_function_registry.h`
- `jit/runtime/shape_function_registry.h`
- `jit/runtime/simple_graph_executor_impl.cpp`
- `jit/runtime/simple_graph_executor_impl.h`
- `jit/runtime/slice_indices_adjust.cpp`
- `jit/runtime/slice_indices_adjust.h`
- `jit/runtime/symbolic_script.cpp`
- `jit/runtime/symbolic_script.h`
- `jit/runtime/symbolic_shape_registry.cpp`
- `jit/runtime/symbolic_shape_registry.h`
- `jit/runtime/symbolic_shape_registry_util.cpp`
- `jit/runtime/symbolic_shape_registry_util.h`
- `jit/runtime/vararg_functions.cpp`
- `jit/runtime/vararg_functions.h`
- `jit/runtime/variable_tensor_list.h`

## Summary

This group contains the symbolic script gradient registry, symbolic shape compute registry, the simple graph executor, and utility functions. The main thread-safety concerns center on the lazy initialization patterns used in both `symbolic_script.cpp` and `symbolic_shape_registry.cpp`, where global maps are lazily populated behind a mutex but then read without holding the lock.

## Issues

### Issue 1
- **Category**: Lazy initialization / global mutable state
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/symbolic_script.cpp`
- **Lines**: 8, 1488-1496, 1610-1639
- **Description**: Global mutable state inside an anonymous namespace: `schema_to_graphs` (map from string to `GradientPair`), `cached_gradient_pairs` (map from schema pointer to `GradientPair`), and `compilation_unit`. `gradientInfoForSchema()` acquires the mutex `lock`, checks if `schema_to_graphs` is empty (triggering lazy load if so), and then reads from both maps while still holding the lock. This is correctly synchronized -- the lock is held for the entire read operation. However, there is no mechanism for concurrent writes to `cached_gradient_pairs` from different callers since the lock is held. This is safe.
- **Fix**: No fix needed. The locking in `gradientInfoForSchema` properly covers both the lazy init and the subsequent reads/writes to the caches.

### Issue 2
- **Category**: Lazy initialization / global mutable state
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/symbolic_shape_registry.cpp`
- **Lines**: 15, 70-77, 353-394
- **Description**: Global mutable maps `cached_schema_to_graph` and `cached_bounded_schema_to_graph`, plus a `compilation_unit`, are lazily initialized. `shapeComputeGraphForSchema()` acquires the mutex `lock`, checks if `cached_schema_to_graph` is empty, and triggers lazy loading if needed. The read of the map also happens while the lock is held. Similarly, `RegisterShapeComputeGraphForSchema()` acquires the lock. This is properly synchronized.
- **Fix**: No fix needed; the lock covers the entire scope of reads and writes.

### Issue 3
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: High
- **File**: `jit/runtime/simple_graph_executor_impl.cpp`
- **Lines**: 14-28
- **Description**: `SimpleGraphExecutorImpl::getPlanFor()` uses `compile_mutex` (inherited from `GraphExecutorImplBase`) to protect the lazy compilation of `execution_plan_`. This is a correct double-checked locking pattern (the lock is acquired, then `execution_plan_` is checked). Thread-safe.
- **Fix**: No fix needed.

### Issue 4
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/symbolic_shape_registry.cpp`
- **Lines**: 58-68
- **Description**: `conditionally_defined_ops()` uses a function-local `static const OperatorMap<std::string>`, which is safely initialized once per C++11 guarantees. Read-only after initialization. Safe.
- **Fix**: No fix needed.

### Issue 5
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: Low
- **File**: `jit/runtime/vararg_functions.cpp`
- **Lines**: 11
- **Description**: `static constexpr int defaultPrecision = 6` is a compile-time constant. No thread-safety concern.
- **Fix**: No fix needed.
