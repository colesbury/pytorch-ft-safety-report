# Free-Threading Safety Audit: inductor_cpp_wrapper

## Overall Risk: MEDIUM

## Files Reviewed
- `inductor/cpp_wrapper/array_ref.h`
- `inductor/cpp_wrapper/common.h`
- `inductor/cpp_wrapper/cpu.h`
- `inductor/cpp_wrapper/cuda.h`
- `inductor/cpp_wrapper/lazy_triton_compile.h`
- `inductor/cpp_wrapper/mps.h`
- `inductor/cpp_wrapper/xpu.h`

## Detailed Analysis

### common.h

Defines `RAIIPyObject` (a simple ref-counted PyObject wrapper), imports AOTI runtime utilities, and provides the `align()` function. The `RAIIPyObject` class properly manages reference counts with `Py_XINCREF`/`Py_XDECREF`. Under free-threading, these are atomic operations, so the class itself is safe.

### lazy_triton_compile.h

This header has **multiple static mutable globals** that are lazily initialized:

```cpp
static PyObject* (*_THPVariable_Wrap)(const at::TensorBase&) = nullptr;
static int32_t (*_THPUtils_unpackInt)(PyObject*) = nullptr;
static PyObject* triton_lazy_compile_module = nullptr;
static PyObject* start_kernel_compile = nullptr;
static PyObject* run_triton_kernel_with_autotune = nullptr;
static PyObject* _module_pending_kernels = nullptr;
```

The `loadLazyCompileFuncs()` function lazily initializes all of these on first call by checking `if (triton_lazy_compile_module == nullptr)`. This is a classic check-then-act race: multiple threads could simultaneously see `nullptr` and attempt to initialize these globals concurrently. The function does Python module imports and attribute lookups without any synchronization beyond the GIL.

Under free-threading, the GIL no longer serializes concurrent calls to `loadLazyCompileFuncs`. While the individual `PyImport_ImportModule` and `PyObject_GetAttrString` calls will use critical sections internally, the overall pattern of "check null, then initialize" is racy for the static pointer variables.

However, these are `static` at file scope in a header, which means each translation unit that includes this header gets its own copy. The generated cpp_wrapper code includes this header once per compiled model. The concern is if the same model's wrapper functions are called concurrently before lazy init completes.

The `startKernelCompile` and `runTritonKernelWithAutotune` functions both call `py::gil_scoped_acquire_simple` before using Python objects, which is correct for free-threading.

### array_ref.h, cpu.h, cuda.h, mps.h, xpu.h

Thin include-forwarding headers. No issues.

## Issues

| # | Severity | Description | Location |
|---|----------|-------------|----------|
| 1 | Medium | Lazy init of static `PyObject*` globals in `loadLazyCompileFuncs()` uses a check-then-act pattern that races under free-threading. Multiple threads could concurrently try to import modules and store results into the same static pointers. | `lazy_triton_compile.h:29-75` |
| 2 | Low | `_module_pending_kernels` is a per-module dict stored as a file-scope static. If two threads in the same translation unit try to access it before initialization, they could race. | `lazy_triton_compile.h:36` |

## Recommendations

1. Use `std::call_once` or `Py_BEGIN_CRITICAL_SECTION`/`Py_END_CRITICAL_SECTION` around the lazy initialization in `loadLazyCompileFuncs()` to ensure it executes exactly once.
2. Since these are `static` in a header (per-TU), the blast radius is limited to concurrent calls within the same compiled model. But the fix is straightforward.

## Summary

The lazy initialization pattern in `lazy_triton_compile.h` is the primary concern. The static `PyObject*` globals are initialized via a check-then-act pattern that was safe under the GIL but becomes racy under free-threading. The actual Python API calls within the functions properly acquire the GIL.
