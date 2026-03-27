# Free-Threading Safety Audit: jit_passes__part11

## Overall Risk: MEDIUM

## Files Audited
- `jit/passes/shape_analysis.h`
- `jit/passes/specialize_autogradzero.cpp`
- `jit/passes/specialize_autogradzero.h`
- `jit/passes/subgraph_rewrite.cpp`
- `jit/passes/subgraph_rewrite.h`
- `jit/passes/symbolic_shape_analysis.cpp`
- `jit/passes/symbolic_shape_analysis.h`
- `jit/passes/symbolic_shape_cache.cpp`
- `jit/passes/symbolic_shape_cache.h`
- `jit/passes/symbolic_shape_runtime_fusion.cpp`
- `jit/passes/symbolic_shape_runtime_fusion.h`
- `jit/passes/tensorexpr_fuser.cpp`
- `jit/passes/tensorexpr_fuser.h`
- `jit/passes/update_differentiable_graph_requires_grad.cpp`
- `jit/passes/update_differentiable_graph_requires_grad.h`

## Summary

Several files contain static mutable boolean flags and one file contains a global mutable cache (the shape cache). The boolean flags are set/read without synchronization, creating potential data races under free-threading. The shape cache uses an internal LRU cache that may or may not be thread-safe depending on the `lazy::Cache` implementation.

## Detailed Analysis

### Issues Found

#### Issue 1: Mutable static `symbolic_shape_analysis_test_mode` (symbolic_shape_analysis.cpp, line 38) -- MEDIUM

```cpp
static bool symbolic_shape_analysis_test_mode = false;

bool setSymbolicShapeAnalysisTestMode(bool value) {
    bool old_value = symbolic_shape_analysis_test_mode;
    symbolic_shape_analysis_test_mode = value;
    return old_value;
}

bool symbolicShapeAnalysisTestModeEnabled() {
    return symbolic_shape_analysis_test_mode;
}
```

A plain `bool` read and written from potentially concurrent threads without synchronization. The read-modify-write in `setSymbolicShapeAnalysisTestMode` is also non-atomic. Should be `std::atomic<bool>`.

#### Issue 2: Mutable static `texpr_reductions_enabled` (tensorexpr_fuser.cpp, line 40) -- MEDIUM

```cpp
static bool texpr_reductions_enabled = false;
```

This is read/written via `setTexprReductionsEnabled`/`texprReductionsEnabled` (declared in header). No synchronization. Should be `std::atomic<bool>`.

#### Issue 3: Mutable static `texpr_fuser_enabled_` (tensorexpr_fuser.cpp, line 149) -- MEDIUM

```cpp
static bool texpr_fuser_enabled_ = true;

void setTensorExprFuserEnabled(bool val) {
    texpr_fuser_enabled_ = val;
}

bool tensorExprFuserEnabled() {
    static const auto enable_opt = c10::utils::get_env("PYTORCH_TENSOREXPR");
    if (!enable_opt.has_value()) {
        return texpr_fuser_enabled_;
    }
    ...
}
```

Same pattern: mutable static bool read/written without synchronization. Should be `std::atomic<bool>`.

#### Issue 4: Global `shapeCache` (symbolic_shape_cache.cpp, line 87) -- MEDIUM

```cpp
ShapeCache shapeCache(kShapeCacheSize);
```

`ShapeCache` is `lazy::Cache<...>`. The `cache_shape_function`, `get_cached_shape_function`, `clear_shape_cache`, and `get_shape_cache_size` functions all access this global cache from potentially any thread. Thread safety depends on the `lazy::Cache` implementation. If `lazy::Cache` is not internally synchronized, this is a data race.

#### Issue 5: Static `OperatorSet` objects (tensorexpr_fuser.cpp, lines 72-100+) -- LOW

```cpp
static OperatorSet _g_custom_operator_set{};  // mutable, returned by ref
static const OperatorSet supported_non_eltwise_set{...};
static const OperatorSet supported_reduction_set{...};
static const OperatorSet supported_misc_set{...};
```

The `_g_custom_operator_set` is mutable and returned by reference via `getCustomOperatorSet()`. If callers modify it concurrently, there is a race. The `const` ones are safe.

### No Issues

- **shape_analysis.h**: Pure declarations.
- **specialize_autogradzero.cpp**: `AutogradZeroSpecializer` is instance-local. The profiling callback at line 42-64 uses `pr->mutex_` for synchronization -- correct.
- **subgraph_rewrite.cpp/h**: `SubgraphRewriter` is an instance-local class. No shared state.
- **symbolic_shape_runtime_fusion.cpp**: Local functions operating on passed-in nodes/graphs.
- **update_differentiable_graph_requires_grad.cpp**: Pure graph traversal.

## Recommendations

1. **symbolic_shape_analysis.cpp**: Change `symbolic_shape_analysis_test_mode` to `std::atomic<bool>`.
2. **tensorexpr_fuser.cpp**: Change `texpr_reductions_enabled` and `texpr_fuser_enabled_` to `std::atomic<bool>`.
3. **symbolic_shape_cache.cpp**: Verify that `lazy::Cache` is thread-safe for concurrent `Add`/`Get`/`Clear` operations. If not, add external synchronization.
4. **tensorexpr_fuser.cpp**: Consider whether `getCustomOperatorSet()` needs synchronization for concurrent access to the returned mutable `OperatorSet`.
