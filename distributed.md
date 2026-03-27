# Free-Threading Safety Audit: distributed

## Overall Risk: Low

## Files Audited
- `distributed/Placement.h`
- `distributed/python_placement.cpp`
- `distributed/python_placement.h`

## Findings

### 1. Module init function `initPlacementBindings` -- Low Risk
**File:** `distributed/python_placement.cpp`
**Lines:** 21-131

This function registers pybind11 classes (`Placement`, `Shard`, `StridedShard`, `Replicate`, `Partial`) onto a Python module. It calls `py::module_::import("torch._opaque_base")` (line 27) and uses `py::reinterpret_borrow` on the passed module (line 22).

This is a module initialization function that is expected to be called once during interpreter setup. Pybind11 class registration is not thread-safe in general, but since this runs during single-threaded module init, it is safe in practice.

No static or global mutable state is introduced. The `placement_class_docstring` on line 11 is a `const` string in an anonymous namespace -- safe.

### 2. Pure C++ data classes in `Placement.h` -- No Issue
**File:** `distributed/Placement.h`

All classes (`Placement`, `Shard`, `StridedShard`, `Replicate`, `Partial`) are plain C++ value types with no static/global state, no Python API calls, and no shared mutable state. They are safe under free-threading.

## Summary

This group contains pybind11 bindings for DTensor placement types and their pure C++ definitions. There is no static mutable state, no direct Python C API usage outside of pybind11, and no lazy initialization patterns. The initialization function follows the standard single-call module init pattern. No free-threading issues were identified.
