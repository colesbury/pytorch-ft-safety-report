# Free-Threading Safety Audit: onnx

**Overall Risk: None**

## Files Reviewed
- `onnx/back_compat.h`
- `onnx/init.cpp`
- `onnx/init.h`
- `onnx/onnx.h`

## Findings

No free-threading issues found.

## Summary

The `onnx` module provides pybind11 bindings for ONNX-related JIT passes and types. Key observations:

- `onnx.h`: Contains only enum definitions and a constexpr constant. No mutable state.
- `back_compat.h`: Contains only constexpr constants for ONNX tensor data type compatibility. No mutable state.
- `init.cpp`: The `initONNXBindings` function registers pybind11 methods and enum bindings. All bound functions are wrappers around JIT pass functions that operate on their arguments. No mutable static or global state is introduced in this file. The `_jit_set_onnx_log_output_stream` lambda creates shared_ptr wrappers around `std::cout`/`std::cerr` and passes them to the ONNX logging subsystem, but the state management is within that subsystem, not in this file.
- `init.h`: Just a function declaration.

All files are either header-only definitions or module initialization code with no mutable global state.
