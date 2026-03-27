# Free-Threading Safety Audit: torch/csrc/utils (Part 3)

## Files Reviewed
- `utils/python_dispatch.h`
- `utils/python_numbers.h`
- `utils/python_raii.h`
- `utils/python_scalars.h`
- `utils/python_strings.h`
- `utils/python_stub.h`
- `utils/python_symnode.cpp`
- `utils/python_symnode.h`
- `utils/python_torch_function_mode.h`
- `utils/python_tuples.h`
- `utils/pythoncapi_compat.h`
- `utils/schema_info.cpp`
- `utils/schema_info.h`
- `utils/six.h`
- `utils/structseq.cpp`

## Overall Risk: Low

## Detailed Analysis

### 1. python_symnode.cpp -- Lazy init in `#else` (non-pybind 2.13) branches

**Risk: Low (conditionally compiled out on modern builds)**

Lines 16-17, 33-34, 50-51, 68-70 each contain a pattern like:

```cpp
static py::handle symint_class =
    py::object(py::module::import("torch").attr("SymInt")).release();
return symint_class;
```

These are classic lazy-init-of-static patterns: the static local `py::handle` is initialized on first call. Under the GIL, only one thread could execute the initialization. Without the GIL, two threads could race on the initialization of the static variable.

However, **C++11 guarantees thread-safe initialization of function-local statics** (magic statics / [stmt.dcl]/4), so the actual initialization itself is not a data race -- the compiler inserts synchronization. The `py::handle` is a trivial non-owning pointer wrapper, and once initialized the value is immutable, so reads are safe.

The real concern is that `py::module::import()` internally calls `PyImport_Import` which requires the GIL (or the import lock in free-threaded Python). But these functions are only invoked from Python-facing code that already holds the GIL. And more importantly, the `#if IS_PYBIND_2_13_PLUS` branch uses `py::gil_safe_call_once_and_store`, which is explicitly designed for this purpose. Modern builds of PyTorch use pybind11 >= 2.13, so the `#else` branches are dead code on current toolchains.

**No action required** for current builds. The `#else` branches are correct enough due to C++11 magic statics. If the `#else` branches are ever exercised without the GIL, the `py::module::import` call needs the GIL held, but callers do hold it.

### 2. python_strings.h -- `PyObject_FastGetAttrString` accesses `tp_getattr`/`tp_getattro`

**Risk: None (pre-3.13 only, correct on 3.13+)**

The `#else` branch (pre-3.13) directly reads `tp->tp_getattr` and `tp->tp_getattro` from the type object. Under free-threading, type object slot reads could race with type mutations. However, the `#if IS_PYTHON_3_13_PLUS` branch correctly delegates to `PyObject_GetOptionalAttrString`, which is the thread-safe API. Since free-threading only exists on Python 3.13+, the unsafe branch is never compiled in a free-threaded build.

### 3. pythoncapi_compat.h -- Third-party compatibility header

**Risk: None**

This is an upstream compatibility shim (`pythoncapi_compat` project). All its polyfills are gated by `PY_VERSION_HEX` checks so that on Python 3.13+ (where free-threading exists), the native CPython implementations are used instead. Patterns like `PyDict_GetItemRef`, `PyWeakref_GetRef`, etc. are only compiled on older Python versions that don't support free-threading.

### No Issues Found in Remaining Files

- **python_dispatch.h**: Pure declarations, no state.
- **python_numbers.h**: Stateless inline conversion functions. No global/static mutable state. All operations are on the passed-in `PyObject*` arguments.
- **python_raii.h**: Template RAII context manager wrappers. Instance-level state only (no statics/globals).
- **python_scalars.h**: Stateless inline conversion functions operating on passed-in arguments.
- **python_stub.h**: Trivial forward declaration of `PyObject`.
- **python_symnode.h**: `PythonSymNodeImpl` class with instance state only. All Python API calls are preceded by `py::gil_scoped_acquire`, which correctly acquires the GIL (or the per-object lock under free-threading).
- **python_torch_function_mode.h**: `StashTorchFunctionModeGuard` operates on TLS (`PythonTorchFunctionTLS`), which is thread-local by definition. No shared mutable state.
- **python_tuples.h**: Stateless inline helpers for packing int64 arrays into new tuples. Uses `PyTuple_SET_ITEM` on freshly created tuples (safe since no other thread can see them yet).
- **schema_info.cpp / schema_info.h**: Pure C++ logic operating on `SchemaInfo` instances. The `static const` vectors in `getTrainingOps()` and `getNonDeterministicOps()` are initialized via C++11 magic statics and are immutable after initialization -- safe. The `static const` `dropout_schema` in `is_nondeterministic()` is also safe for the same reason. No Python API usage.
- **six.h**: Stateless inline helpers. `isStructSeq` calls `pybind11::type::handle_of()` and attribute access, but these are read-only type introspection operations.
- **structseq.cpp**: `returned_structseq_repr` operates on a single `PyStructSequence` object. It reads `Py_TYPE(obj)`, `Py_SIZE(obj)`, `tp_name`, and `tp_members`, all of which are type-level metadata that is stable for struct sequences. `PyTuple_GetItem` returns a borrowed reference from `tup`, but `tup` is a local variable that no other thread can access.

## Summary

The files in this batch are overwhelmingly safe for free-threading. The only pattern worth noting is the lazy static initialization in `python_symnode.cpp`'s `#else` branches, but these are (a) protected by C++11 magic statics and (b) not compiled on modern pybind11 builds where the explicitly thread-safe `gil_safe_call_once_and_store` is used instead. No actionable issues were found.
