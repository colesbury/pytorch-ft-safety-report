# Free-Threading Safety Audit: jit_passes__part3

**Overall Risk: LOW-MEDIUM**

Most files are pure C++ graph transformations with no Python C API usage. The main concerns are: (1) a lazily-initialized static singleton in `device_type_analysis.cpp` that is not thread-safe, (2) a `static CompilationUnit` in `decompose_ops.cpp` with potential lazy initialization concerns, and (3) static `OperatorSet` objects in `decompose_ops.cpp` and `frozen_conv_folding.cpp` (from the next group, but the pattern first appears here).

---

## File Analysis

### jit/passes/create_functional_graphs.cpp, .h
- **Risk: NONE**
- The `FunctionalGraphSlicer` is created per-invocation. All state is local. No static/global mutable state. No Python C API.

### jit/passes/dead_code_elimination.cpp, .h
- **Risk: NONE**
- `DeadCodeEliminator` is created per-invocation with all state local. No static/global mutable state. No Python C API.

### jit/passes/decompose_ops.cpp, .h
- **Risk: MEDIUM**
- **`static CompilationUnit decompose_funcs(R"SCRIPT(...)SCRIPT")` (line 195):** This is a function-local static inside `DecomposeOps(std::shared_ptr<Graph>&)`. In C++11 and later, function-local static initialization is thread-safe (guaranteed by the standard -- "magic statics"). Once initialized, the `CompilationUnit` is read-only (its graphs are accessed but not mutated by the pass). **Safe for concurrent access after initialization.**
- **`static const OperatorSet decomposable_normalization_ops = {...}` (line 35):** Function-local static const, thread-safe initialization. Read-only after init.
- `static RegisterOperators reg_ops(...)` (line 60): Static init registration of custom operators. Runs at load time.
- The pass itself mutates the input graph, but the graph is not shared across threads (callers are responsible for this).
- **Potential concern:** If `DecomposeOps` is called concurrently for the first time, the `static CompilationUnit` construction involves JIT compilation of TorchScript code. This itself may involve Python interactions (operator lookups, etc.) depending on the implementation. However, since C++ guarantees single-initialization of function-local statics, only one thread will perform the initialization while others block. This is safe but could be a performance bottleneck.

### jit/passes/device_type_analysis.cpp, .h
- **Risk: MEDIUM**
- **`static std::unique_ptr<OperatorMap<PropRule>> device_prop_registry_ = nullptr` (line 248):** This is a static class member of `DeviceTypePropagationPass` initialized to `nullptr`. It is lazily initialized in `buildRuleRegistry()` (line 218-241) using a check-then-assign pattern:
  ```cpp
  if (device_prop_registry_)
    return;
  static OperatorMap<PropRule> temp_registry{...};
  device_prop_registry_ = std::make_unique<OperatorMap<PropRule>>(std::move(temp_registry));
  ```
  **Issue:** This is a classic data race under free-threading. If two threads call `DeviceTypePropagation` simultaneously for the first time, both may see `device_prop_registry_` as null and attempt to initialize it concurrently, or one may see a partially-constructed object.

  Note: The `temp_registry` is itself a function-local static, so its initialization is thread-safe. But `device_prop_registry_` is a file-scope static `unique_ptr`, and the assignment `device_prop_registry_ = std::make_unique<...>(std::move(temp_registry))` is not atomic. The `std::move` from `temp_registry` would also leave `temp_registry` in a moved-from state, making subsequent initializations from other threads incorrect even if the `unique_ptr` assignment were atomic.

  **Recommendation:** Convert to a Meyers singleton (`static OperatorMap<PropRule>& getRegistry() { static OperatorMap<PropRule> reg{...}; return reg; }`) or use `std::call_once`.

### jit/passes/dtype_analysis.cpp, .h
- **Risk: NONE**
- The `DtypePropagationPass` creates a new `dtype_prop_registry_` per instance (line 312: `dtype_prop_registry_ = std::make_unique<...>()`). No shared static state. No Python C API.

### jit/passes/eliminate_no_ops.cpp, .h
- **Risk: NONE**
- Pure graph transformation with all state local. No static/global mutable state.

### jit/passes/erase_number_types.cpp, .h
- **Risk: NONE**
- Pure graph transformation. No static/global mutable state. No Python C API.

### jit/passes/fixup_trace_scope_blocks.cpp, .h
- **Risk: NONE**
- All helper structs (`ConvertTracedAttrReferences`, `MakeDefsDominateUses`) are created per-invocation with local state. The free functions operate on graphs passed by reference. No static/global mutable state. No Python C API.

---

## Summary of Issues

| File | Issue | Severity | Description |
|------|-------|----------|-------------|
| device_type_analysis.cpp | Lazy init of static `device_prop_registry_` | Medium | Classic check-then-assign race on file-scope static `unique_ptr`. Two concurrent first calls can race. |
| decompose_ops.cpp | Static `CompilationUnit` | Low | C++ guarantees thread-safe init of function-local statics. Safe but could serialize first-time callers. |

## Recommendations

1. **device_type_analysis.cpp (HIGH PRIORITY):** The lazy initialization of `device_prop_registry_` is not thread-safe. Fix by converting to a function-local static (Meyers singleton pattern):
   ```cpp
   static OperatorMap<PropRule>& getDevicePropRegistry() {
     static OperatorMap<PropRule> registry{
       {"aten::cpu(Tensor self) -> Tensor", setReturnstoDeviceRule(DeviceType::CPU)},
       // ... other entries ...
     };
     return registry;
   }
   ```
   Then use `getDevicePropRegistry()` directly instead of the `device_prop_registry_` member. This leverages C++'s thread-safe function-local static initialization.

2. **decompose_ops.cpp:** No action needed. The `static CompilationUnit` uses C++ magic static initialization which is already thread-safe. Performance-wise, the first concurrent call may block, but correctness is guaranteed.
