# Free-Threading Safety Audit: cpu

**Overall Risk: None**

## Files Reviewed
- `cpu/Module.cpp`
- `cpu/Module.h`

## Findings

No free-threading issues found.

## Summary

The `cpu` module is a minimal pybind11 wrapper that exposes two functions: `_init_amx` and `_get_cpu_capability`. Both call into ATen functions and return values without using any mutable static or global state. The `initModule` function is called once during module initialization. All bindings go through pybind11 which manages the GIL.
