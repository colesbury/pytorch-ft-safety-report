# Free-Threading Safety Audit: inductor_aoti_eager

## Overall Risk: LOW

## Files Reviewed
- `inductor/aoti_eager/kernel_holder.cpp`
- `inductor/aoti_eager/kernel_holder.h`
- `inductor/aoti_eager/kernel_meta_info.cpp`
- `inductor/aoti_eager/kernel_meta_info.h`

## Detailed Analysis

### AOTIPythonKernelHolder (kernel_holder.cpp / kernel_holder.h)

This class implements a per-operator kernel cache for AOTI eager mode. Key observations:

**GIL-protected Python interaction**: The two methods that call into Python (`init_aoti_kernel_cache` and `produce_aoti_kernel_lib`) both acquire the GIL via `py::gil_scoped_acquire`. Under free-threading, pybind11's `gil_scoped_acquire` maps to the appropriate critical section mechanisms, so the Python API calls within those functions are safe.

**Instance-level mutable state (aoti_kernel_cache_)**: The `aoti_kernel_cache_` vector is mutated in `cache_miss` (push_back) and read in `cache_lookup`. However, `AOTIPythonKernelHolder` is an `OperatorKernel` instance owned by the dispatcher. Each dispatch key gets its own instance. The dispatcher's boxing infrastructure serializes calls through a given operator kernel, and these kernel holders are per-operator, so concurrent access to the same `AOTIPythonKernelHolder` instance from multiple threads would only happen if the same op on the same dispatch key is called concurrently. The current architecture does not protect against this with a mutex, but this is an existing design constraint of the eager AOTI path (it was already not designed for concurrent use of the same kernel holder).

This is not a free-threading regression -- it was already unsafe under the GIL since `operator()` releases the GIL when calling into the AOTI model runner.

**getAOTIModelRunnerRegistry**: Accessed in `load_aoti_model_runner` (read-only at runtime). The registry is populated at static-init time via `RegisterAOTIModelRunner`. Safe.

### kernel_meta_info.cpp / kernel_meta_info.h

Pure data structures (`TensorMetadata`, `ParameterMetadata`) with comparison operators. No static or global mutable state. No Python C API. Entirely safe.

## Summary

No new free-threading issues introduced beyond pre-existing design constraints. The Python interactions properly acquire the GIL. The instance-level cache was already not thread-safe for concurrent same-operator calls, which is a pre-existing limitation unrelated to the GIL.
