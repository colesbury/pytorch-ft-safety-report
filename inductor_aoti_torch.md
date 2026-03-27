# Free-Threading Safety Audit: inductor_aoti_torch

## Overall Risk: LOW

## Files Reviewed
- `inductor/aoti_torch/generated_enum_converters.h`
- `inductor/aoti_torch/mkldnn_tensor.cpp`
- `inductor/aoti_torch/mkldnn_tensor.h`
- `inductor/aoti_torch/oss_proxy_executor.cpp`
- `inductor/aoti_torch/oss_proxy_executor.h`
- `inductor/aoti_torch/proxy_executor.h`
- `inductor/aoti_torch/shim_common.cpp`
- `inductor/aoti_torch/shim_cpu.cpp`
- `inductor/aoti_torch/shim_cuda.cpp`
- `inductor/aoti_torch/shim_mps.cpp`
- `inductor/aoti_torch/shim_xpu.cpp`
- `inductor/aoti_torch/tensor_converter.cpp`
- `inductor/aoti_torch/tensor_converter.h`
- `inductor/aoti_torch/utils.h`

## Detailed Analysis

### shim_common.cpp

This is the main C ABI shim layer between the AOTInductor runtime and ATen. It implements the `aoti_torch_*` functions declared in the C headers. All functions are stateless: they receive handles, convert them to ATen types, call ATen functions, and return results.

The device type and dtype functions (`aoti_torch_device_type_cpu`, `aoti_torch_dtype_float32`, etc.) return constant integers. No mutable state.

No Python C API. No static mutable state. Safe.

### shim_cpu.cpp, shim_cuda.cpp, shim_mps.cpp, shim_xpu.cpp

Device-specific shim functions following the same pattern as `shim_common.cpp`. Each function converts handles to ATen types, calls native functions, and returns results. No static mutable state.

The CUDA shim includes `aoti_torch_cuda_caching_allocator_raw_alloc`/`raw_delete` which delegate to the CUDA caching allocator, which has its own internal thread safety.

No issues.

### oss_proxy_executor.cpp / oss_proxy_executor.h

`OSSProxyExecutor` is constructed once per model load. It parses a JSON file to build a list of `OSSOpKernel` objects. At runtime, `call_function` indexes into `op_kernels_` and calls boxed dispatcher functions.

The instance state (`op_kernels_`, `device_`, `custom_objs_`) is set at construction time and only read at runtime. The `call_function` method does modify the local `op_kernel->stack_` vector, but each call constructs a fresh stack from the kernel's template. This is safe as long as `call_function` is not called concurrently on the same `OSSProxyExecutor` for the same `extern_node_index` -- which is guaranteed by the model container's locking.

No Python C API. No shared mutable static state.

### tensor_converter.cpp / tensor_converter.h

`unsafe_alloc_new_handles_from_tensors` and `alloc_tensors_by_stealing_from_handles` are pure functional utilities with no state. Safe.

### utils.h

Template utilities for converting between C ABI types and ATen types. No state. Safe.

### mkldnn_tensor.cpp / mkldnn_tensor.h

Thin wrappers around MKL-DNN tensor operations. No state. Safe.

### proxy_executor.h

Abstract base class with a pure virtual `call_function` method. No state. Safe.

## Summary

This group implements the C ABI shim layer between the AOTInductor runtime and ATen. All functions are stateless, taking handles in and returning handles out. No Python C API usage, no mutable static/global state. No free-threading issues.
