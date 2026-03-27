# Free-Threading Safety Audit: inductor_static_launcher

## Overall Risk: LOW

## Files Reviewed
- `inductor/static_launcher/cuda.cpp`
- `inductor/static_launcher/cuda.h`
- `inductor/static_launcher/xpu.cpp`
- `inductor/static_launcher/xpu.h`

## Detailed Analysis

### cuda.cpp

Implements a static launcher for triton-compiled CUDA kernels, exposed as Python static methods on `_StaticCudaLauncher`.

**Python C API usage**: The `load_kernel` and `launch_kernel` functions are `METH_VARARGS` Python methods that use `PyArg_ParseTuple`, `PyTuple_GetItem`, `PyObject_GetAttrString`, `PyObject_Call`, and `Py_BuildValue`. Under free-threading:

- `PyArg_ParseTuple`: Safe (reads from args tuple, which is owned by the calling frame).
- `PyTuple_GetItem`: Returns a borrowed reference from the args tuple. Since the tuple is alive for the duration of the call (held by the caller's stack frame), the borrowed reference remains valid. Safe.
- `PyObject_GetAttrString`, `PyObject_Call`: These perform attribute lookups and calls. Under free-threading, Python internally uses critical sections for these. Safe.
- `Py_BuildValue`, `Py_RETURN_NONE`: Safe.

**Static type object**: `StaticCudaLauncherType` is a `PyTypeObject` defined at file scope. It is fully initialized by `StaticCudaLauncher_init` (called during module init, which is single-threaded). After initialization, it is only read. Safe.

**`StaticCudaLauncherMethods`**: A `std::array<PyMethodDef, 2>` at file scope, initialized statically. Immutable after program start. Safe.

**`getPointer` helper**: Uses `THPUtils_checkLong`, `THPUtils_unpackUInt64`, `PyObject_GetAttrString`, `PyObject_Call`. All Python C API calls that are safe under free-threading (they operate on local references or use internal critical sections). The borrowed reference from `PyTuple_GetItem` in `parseKernelArgs` is safe since the tuple is alive throughout.

**No mutable static/global state** beyond the type object (initialized once).

### xpu.cpp

Follows the exact same architecture as `cuda.cpp` but for XPU/SYCL kernels. Uses `PyCapsule` for kernel lifetime management. Same Python C API patterns.

**`StaticXpuLauncherType`**: Same pattern as CUDA -- initialized once during module init, then read-only. Safe.

**`PyCapsule_New` / `PyCapsule_GetPointer`**: These are safe under free-threading (capsules are simple ref-counted objects with a destructor).

## Summary

Both launchers use Python C API correctly for the free-threading model. The borrowed references from `PyTuple_GetItem` are safe because the source tuples remain alive. The static type objects and method definitions are initialized once during module init and are immutable at runtime. No mutable static/global state is accessed at runtime. No free-threading issues.
