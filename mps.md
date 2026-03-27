# Free-Threading Safety Audit: mps

**Overall Risk: None**

## Files Reviewed
- `mps/Module.cpp`
- `mps/Module.h`

## Findings

No free-threading issues found.

## Summary

The `mps` module provides Python bindings for MPS (Metal Performance Shaders) device operations. All functions are thin wrappers that delegate to `at::detail::getMPSHooks()` or ATen MPS APIs. No mutable static or global state is used in the Python-facing code. The `_MPSModule_methods` array and `initModule` function are both set up during module initialization. The MPS Metal shader library and kernel function classes managed via pybind11 use `std::shared_ptr` ownership and do not maintain global mutable state in this file.
