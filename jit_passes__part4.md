# Free-Threading Safety Audit: jit_passes__part4

**Overall Risk: LOW**

This group contains module-level freezing and optimization passes. Most are pure C++ graph transformations with no Python C API interactions. There is one notable pattern: the `frozen_conv_add_relu_fusion.cpp` uses a static `std::function` for runtime registration of a GPU implementation, which has a benign initialization race pattern.

---

## File Analysis

### jit/passes/fixup_trace_scope_blocks.h
- **Risk: NONE**
- Header-only declaration.

### jit/passes/fold_conv_bn.cpp, .h
- **Risk: NONE**
- The `FoldConvBatchNormHelper` class is created per-invocation. `computeUpdatedConvWeightAndBias` is a pure function. Module iteration and pattern matching operate on module structures. No static/global mutable state. No Python C API.
- `addBiasForConvIfNone` mutates the module, but this is expected to be called in a single-threaded context (module freezing/optimization pipeline).

### jit/passes/fold_linear_bn.cpp, .h
- **Risk: NONE**
- `computeUpdatedLinearWeightAndBias` is a pure function operating on input tensors. No state at all.

### jit/passes/freeze_module.cpp, .h
- **Risk: NONE**
- The `AttributePropagator` class is created per-invocation of `freeze_module` or `freeze_module_inplace`. All state is local to the instance. The `module_` reference is the cloned or user-provided module -- callers are responsible for ensuring no concurrent access to the same module.
- Helper functions (`splitName`, `concatName`, `checkModuleDoesNotReturnSelf`) are all pure/local.
- No static/global mutable state. No Python C API.

### jit/passes/frozen_concat_linear.cpp, .h
- **Risk: NONE**
- `ConcatLinearLayers` is created per-invocation with local state. No static/global mutable state. No Python C API.

### jit/passes/frozen_conv_add_relu_fusion.cpp, .h
- **Risk: LOW**
- **`static std::function<void(std::shared_ptr<Graph>&)> impl` (line 11 in .cpp):** A file-scope static `std::function` returned by reference from `getFuseFrozenConvAddReluImpl()`. This is used as a registration point: the CUDA-side implementation sets it.
- **Initialization:** The function is initialized to an empty `std::function` (default-constructed). Its value is set by the `dummyInitializer` lambda in `frozen_conv_add_relu_fusion_cuda.cpp` (line 124-127), which runs at static init time (before `main`).
- **Analysis:** Since the assignment happens at static initialization time (dynamic initializer of `dummyInitializer`), and queries happen after `main()` starts (during graph optimization), there is no data race under normal execution. The static init ordering between the two TUs is not an issue because the write happens in one TU's static init and reads happen at runtime.
- **Caveat:** If a shared library containing the CUDA implementation is loaded dynamically at runtime (e.g., via `dlopen`) while another thread is calling `FuseFrozenConvAddRelu`, there could be a race. However, this is a general shared library loading concern, not specific to free-threading.

### jit/passes/frozen_conv_add_relu_fusion_cuda.cpp
- **Risk: LOW**
- Contains the actual implementation `fuseFrozenConvAddReluImpl` (file-scope anonymous namespace function) and registers it via:
  ```cpp
  auto dummyInitializer = []() {
    getFuseFrozenConvAddReluImpl() = fuseFrozenConvAddReluImpl;
    return true;
  }();
  ```
  This runs at static initialization time. See analysis above.
- The implementation itself uses `SubgraphRewriter` and `graph_rewrite_helper` utilities, all operating on the passed graph. No additional static state.
- `static const OperatorSet` objects inside `supportedAddOrSub` and `supportedMulOrDiv` in `frozen_conv_folding.cpp` use function-local statics -- thread-safe initialization.

### jit/passes/frozen_conv_folding.cpp, .h
- **Risk: NONE**
- `FoldFrozenConvBatchnorm`, `FoldFrozenConvAddOrSub`, `FoldFrozenConvMulOrDiv` are all pure graph transformations operating on blocks. All state is stack-local.
- Function-local `static const OperatorSet` objects (lines 137-143, 295-302) use C++ magic statics for thread-safe initialization and are immutable after init.

### jit/passes/frozen_graph_optimizations.cpp, .h
- **Risk: NONE**
- `OptimizeFrozenGraph` is a simple composition of other passes. No state. No Python C API.

---

## Summary of Issues

| File | Issue | Severity | Description |
|------|-------|----------|-------------|
| frozen_conv_add_relu_fusion.cpp | Static `std::function` for GPU impl registration | Low | Written at static init time, read at runtime. No race under normal execution. |
| frozen_conv_add_relu_fusion_cuda.cpp | Static initializer writes to above | Low | Runs at load time before any reads. Safe. |

## Recommendations

1. **frozen_conv_add_relu_fusion.cpp:** The current pattern is safe for normal execution. If there is concern about dynamic library loading at runtime, the `std::function` could be protected with `std::atomic` flag or `std::call_once`. However, this is very low priority since PyTorch's GPU support libraries are loaded before JIT optimization runs.

2. **No other changes needed.** All other files in this group use per-invocation local state and are safe under free-threading.

No Python C API usage was found in any file in this group.
