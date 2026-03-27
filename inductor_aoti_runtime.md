# Free-Threading Safety Audit: inductor_aoti_runtime

## Overall Risk: LOW

## Files Reviewed
- `inductor/aoti_runtime/arrayref_tensor.h`
- `inductor/aoti_runtime/constant_type.h`
- `inductor/aoti_runtime/device_utils.h`
- `inductor/aoti_runtime/interface.h`
- `inductor/aoti_runtime/kernel_context_tls.h`
- `inductor/aoti_runtime/mini_array_ref.h`
- `inductor/aoti_runtime/model.h`
- `inductor/aoti_runtime/model_base.h`
- `inductor/aoti_runtime/model_container.h`
- `inductor/aoti_runtime/scalar_to_tensor.h`
- `inductor/aoti_runtime/sycl_runtime_wrappers.h`
- `inductor/aoti_runtime/thread_local.h`
- `inductor/aoti_runtime/utils.h`
- `inductor/aoti_runtime/utils_cuda.h`
- `inductor/aoti_runtime/utils_xpu.h`

## Detailed Analysis

This group implements the AOTInductor runtime that is compiled into the generated model `.so` files. It does NOT interact with the Python C API.

### model_container.h (AOTInductorModelContainer)

The container has its own thread-safety mechanisms:
- `model_exec_mutex_` (shared_mutex) protects model execution vs. constant swapping
- `models_mutex_` protects `available_models_` and `pending_models_`
- `pending_models_available_` condition variable for model recycling

These are C++ synchronization primitives that work correctly regardless of the GIL. The container was designed for multi-threaded use from the start.

### kernel_context_tls.h

Uses `thread_local KernelContext* tls_kernel_context` for per-thread kernel context. Thread-local storage is inherently thread-safe.

### CACHE_TORCH_DTYPE / CACHE_TORCH_DEVICE / CACHE_TORCH_LAYOUT / CACHE_TORCH_MEMORY_FORMAT macros (utils.h)

These use `static auto` for caching constant values like dtype IDs:
```cpp
#define CACHE_TORCH_DTYPE(typename) \
  static auto cached_torch_dtype_##typename = aoti_torch_dtype_##typename()
```

These are function-local statics that call functions returning constant integer values. C++11 guarantees thread-safe initialization of function-local statics, and the values are immutable after initialization. Safe.

### model_base.h

Contains platform-specific implementations (mmap, dladdr for Windows). All are pure C/C++ with no Python interaction. The model base class manages constants loading from binary blobs and device memory allocation. No shared mutable static state.

### thread_local.h

`ThreadLocalCachedOutputTensor` and `ThreadLocalCachedOutputArray` are designed for thread-local caching of output tensors. They hold per-thread storage and are safe by design.

### utils.h

RAII wrappers (`RAIIAtenTensorHandle`, `MaybeOwningAtenTensorHandle`, `ConstantHandle`, etc.) for tensor handle lifetime management. Pure value types with no shared state.

## Summary

This group is entirely C++ runtime code compiled into model `.so` files, with no Python C API interaction. It has its own proper C++ synchronization for multi-threaded model execution. Thread-local storage and function-local statics are used correctly. No free-threading issues.
