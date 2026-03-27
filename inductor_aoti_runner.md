# Free-Threading Safety Audit: inductor_aoti_runner

## Overall Risk: LOW

## Files Reviewed
- `inductor/aoti_runner/model_container_runner.cpp`
- `inductor/aoti_runner/model_container_runner.h`
- `inductor/aoti_runner/model_container_runner_cpu.cpp`
- `inductor/aoti_runner/model_container_runner_cpu.h`
- `inductor/aoti_runner/model_container_runner_cuda.cpp`
- `inductor/aoti_runner/model_container_runner_cuda.h`
- `inductor/aoti_runner/model_container_runner_mps.cpp`
- `inductor/aoti_runner/model_container_runner_mps.h`
- `inductor/aoti_runner/model_container_runner_xpu.cpp`
- `inductor/aoti_runner/model_container_runner_xpu.h`
- `inductor/aoti_runner/pybind.cpp`
- `inductor/aoti_runner/pybind.h`

## Detailed Analysis

### Global Registry: getAOTIModelRunnerRegistry

```cpp
std::unordered_map<std::string, CreateAOTIModelRunnerFunc>&
getAOTIModelRunnerRegistry() {
  static std::unordered_map<std::string, CreateAOTIModelRunnerFunc>
      aoti_model_runner_registry_;
  return aoti_model_runner_registry_;
}
```

This registry is populated at static-init time by `RegisterAOTIModelRunner` instances (e.g., `register_cpu_runner`, `register_cuda_runner`, `register_xpu_runner`, `register_mps_runner`). The comment in the header states: "It is not thread-safe. Because it is expected to be called during the initialization of the program."

At runtime, the registry is only read (lookups in `AOTIModelPackageLoader` and `AOTIPythonKernelHolder`). Static initialization is single-threaded, so writes complete before any reads. Safe.

### AOTIModelContainerRunner

This is a pure C++ class that loads a `.so` file, resolves function pointers, and calls them. It has no Python C API usage. All mutable state is instance-level (the container handle, function pointers, etc.), so thread safety depends on the caller not sharing a single runner across threads without synchronization -- which is the intended usage pattern (the model container itself has internal locking).

### Device-specific runners (CPU, CUDA, MPS, XPU)

Thin subclasses that delegate to `AOTIModelContainerRunner`. The CUDA and XPU variants override `run_impl` to inject the current device stream. No static mutable state. No Python C API.

### pybind.cpp

`initAOTIRunnerBindings` registers pybind11 bindings for all runner types. Called once during module init. The lambda functions that handle `unsafe_alloc_void_ptrs_from_tensors` etc. are pure C++/ATen operations exposed via pybind11.

No issues.

## Summary

No free-threading issues. The global registry follows a safe write-at-init, read-at-runtime pattern. All runner classes are pure C++ with no Python C API and no shared mutable static state.
