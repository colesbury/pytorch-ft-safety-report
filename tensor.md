# Free-Threading Safety Audit: tensor

**Overall Risk: Medium**

## Files Reviewed
- `tensor/python_tensor.cpp`
- `tensor/python_tensor.h`

## Findings

### 1. Mutable Static `default_backend` Without Synchronization
**Severity: Medium**
**Location:** `tensor/python_tensor.cpp`, line 61

```cpp
static Backend default_backend = Backend::CPU;
```

The `default_backend` variable is read by `get_default_dispatch_key()`, `get_default_device()`, and `set_default_storage_type()`, and written by `set_default_tensor_type()` (line 344). These functions are all callable from Python. Under free-threading, concurrent reads and writes to `default_backend` constitute a data race.

Affected functions:
- `set_default_tensor_type()` (line 318) -- writes `default_backend`
- `get_default_dispatch_key()` (line 457) -- reads `default_backend`
- `get_default_device()` (line 461) -- reads `default_backend`
- `set_default_storage_type()` (line 306) -- reads `default_backend`

### 2. Mutable Static `tensor_types` Vector
**Severity: Low**
**Location:** `tensor/python_tensor.cpp`, line 304

```cpp
static std::vector<PyTensorType*> tensor_types;
```

The `tensor_types` vector is populated during `initialize_aten_types()` at init time and then read by `py_bind_tensor_types()`, `PyTensorType_Check()`, and `initialize_python_bindings()`. After initialization, the vector is only read, so this is safe as long as initialization completes before any reader accesses it. Since `initialize_python_bindings()` is called during module init (before other threads can access it), this should be safe in practice.

### 3. Python Object Operations in `set_default_tensor_type` and Related Functions
**Severity: Low**
**Location:** `tensor/python_tensor.cpp`, lines 306-346

The `set_default_storage_type` function (line 306) calls `PyImport_ImportModule` and `PyObject_SetAttrString` which manipulate Python module state. Under free-threading, if `py_set_default_tensor_type` or `py_set_default_dtype` is called concurrently from multiple threads, the sequence of operations (set storage type, then set default dtype, then set default backend) is not atomic. This could leave the system in an inconsistent state where the storage type, dtype, and backend don't match.

## Summary

The primary concern is the `default_backend` static variable which is read and written without synchronization. This variable is accessed by functions like `get_default_dispatch_key()` and `get_default_device()` which may be called frequently. While setting the default tensor type is not a common operation in typical code, the lack of synchronization means data races are theoretically possible. The `tensor_types` vector is safe since it's only mutated during initialization. Consider making `default_backend` an `std::atomic<Backend>` or protecting access with a mutex.
