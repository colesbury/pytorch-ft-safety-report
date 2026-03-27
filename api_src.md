# Free-Threading Safety Audit: api_src

**Overall Risk: None**

## Files Reviewed
- `api/src/cuda.cpp`
- `api/src/enum.cpp`
- `api/src/imethod.cpp`
- `api/src/jit.cpp`
- `api/src/mps.cpp`
- `api/src/serialize.cpp`
- `api/src/xpu.cpp`

## Summary

No free-threading issues found. These files are part of the pure C++ frontend (libtorch) and do not interact with the Python C API.

- **cuda.cpp, mps.cpp, xpu.cpp**: Device query and seed-setting functions. Generator access is already protected by `std::lock_guard<std::mutex>` on the generator mutex. No static mutable state. No Python objects.
- **enum.cpp**: Macro-generated enum definitions. No mutable state.
- **imethod.cpp**: Contains a lazy-init pattern (`isArgumentNamesInitialized_` / `argumentNames_`) but these are `mutable` instance members on `IMethod`, not static/global state. Thread safety of a single `IMethod` instance is the caller's responsibility (typical C++ object model). This is not a Python-specific free-threading concern.
- **jit.cpp**: Simple factory function creating a new `CompilationUnit`. No shared mutable state.
- **serialize.cpp**: Thin wrappers around `jit::pickle_save`/`jit::pickle_load`. No mutable state.
