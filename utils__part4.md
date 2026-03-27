# Free-Threading Safety Audit: utils/ (Part 4)

## Files Reviewed
- `torch/csrc/utils/structseq.h`
- `torch/csrc/utils/tensor_apply.cpp`
- `torch/csrc/utils/tensor_apply.h`
- `torch/csrc/utils/tensor_dtypes.cpp`
- `torch/csrc/utils/tensor_dtypes.h`
- `torch/csrc/utils/tensor_flatten.cpp`
- `torch/csrc/utils/tensor_flatten.h`
- `torch/csrc/utils/tensor_layouts.cpp`
- `torch/csrc/utils/tensor_layouts.h`
- `torch/csrc/utils/tensor_list.cpp`
- `torch/csrc/utils/tensor_list.h`
- `torch/csrc/utils/tensor_memoryformats.cpp`
- `torch/csrc/utils/tensor_memoryformats.h`
- `torch/csrc/utils/tensor_new.cpp`
- `torch/csrc/utils/tensor_new.h`

## Overall Risk: Medium

## Issues

### 1. Global mutable `memory_format_registry` without synchronization
**File:** `torch/csrc/utils/tensor_memoryformats.cpp:13-14`
**Severity:** Low (init-time only writes, read-after-init)
**Description:**
`memory_format_registry` is a file-static `std::array<PyObject*, ...>` that is written during `initializeMemoryFormats()` and read via `getTHPMemoryFormat()`. Under the GIL, concurrent reads were safe. The writes happen during module initialization, which is typically single-threaded. The reads after init return borrowed `PyObject*` pointers, which is safe as the objects are intentionally leaked (never deallocated). This is a write-once-read-many pattern and is safe provided `initializeMemoryFormats()` completes before any concurrent calls to `getTHPMemoryFormat()`. Python module init ordering should guarantee this, but there is no explicit memory barrier.

```cpp
std::array<PyObject*, static_cast<int>(at::MemoryFormat::NumOptions)>
    memory_format_registry = {};
```

**Recommendation:** Low priority. If defensive hardening is desired, the array elements could be made `std::atomic<PyObject*>` to ensure visibility across threads, but the init-before-use guarantee from the Python import system should suffice.

### 2. `static PythonArgParser` instances in `tensor_new.cpp` (6 instances)
**File:** `torch/csrc/utils/tensor_new.cpp` lines 611, 717, 1291, 1334, 1405, 1539
**Severity:** Low
**Description:**
Multiple `static PythonArgParser` objects are constructed in function-local scope. These are initialized on first call and then reused. If `PythonArgParser` construction and `parse()` calls are thread-safe (e.g., the parser is immutable after construction, and C++11 guarantees thread-safe initialization of function-local statics), this is fine. However, `PythonArgParser` may cache Python type objects or maintain internal mutable state. Whether this is a real issue depends on `PythonArgParser`'s implementation.

**Recommendation:** Verify that `PythonArgParser` is safe for concurrent use from multiple threads. If it has mutable internal state accessed during `parse()`, it needs synchronization.

### 3. `static std::string sig` in template function
**File:** `torch/csrc/utils/tensor_new.cpp:1385`
**Severity:** Medium
**Description:**
In `_validate_sparse_compressed_tensor_args_template`, a `static std::string sig` is declared and written to inside a switch statement. Because this is a template function, each instantiation gets its own static variable, but C++11 function-local static initialization is only thread-safe for the *first* initialization. After the first call initializes `sig` to empty, every subsequent call re-assigns it inside the switch. This is a data race: multiple threads can concurrently write to and read from `sig`.

```cpp
template <c10::Layout required_layout>
static void _validate_sparse_compressed_tensor_args_template(...) {
  ...
  static std::string sig;
  switch (required_layout) {
    case c10::Layout::SparseCsr:
      sig = "_validate_sparse_csr_tensor(...)";
      break;
    ...
  }
  static PythonArgParser parser({sig});
  ...
}
```

The `static PythonArgParser parser({sig})` is also problematic: it captures `sig` by reference at construction time (first call only), so the parser is built from whatever `sig` was on the first call. Subsequent calls write to `sig` but the parser is already initialized, so the write is wasted but still constitutes a race.

**Recommendation:** Make `sig` a `constexpr` compile-time string or a plain local (non-static) variable. Since the template parameter `required_layout` is a compile-time constant, a `constexpr` approach works. Alternatively, use `if constexpr` to return a `const char*` literal.

### 4. `PySequence_Fast_ITEMS` returns borrowed references to container items
**File:** `torch/csrc/utils/tensor_new.cpp:249`
**Severity:** Low
**Description:**
`PySequence_Fast_ITEMS` returns a pointer to the internal `PyObject**` array of a list/tuple. Under free-threading, if another thread mutates the list concurrently, these borrowed references could become dangling. However, in this context the `seq` object was just created via `PySequence_Fast(obj, ...)` which creates a new tuple/list, so the caller owns it exclusively. This is safe as long as `obj` is not being mutated to alias `seq` somehow.

**Recommendation:** No action needed. The freshly-created `seq` is exclusively owned.

### 5. Init-time global state population in `tensor_dtypes.cpp` and `tensor_layouts.cpp`
**File:** `torch/csrc/utils/tensor_dtypes.cpp:9-38`, `torch/csrc/utils/tensor_layouts.cpp:18-52`
**Severity:** Low (init-time only)
**Description:**
`initializeDtypes()` and `initializeLayouts()` populate global registries (`dtype_registry` and `layout_registry` in `DynamicTypes.cpp`) and add objects to the `torch` module. These functions are called during module initialization, which Python's import machinery serializes. Under free-threading, Python still holds the import lock during module init, so concurrent writes should not occur.

**Recommendation:** No action needed as long as these functions are only called from module init.

## Files With No Issues

- **`structseq.h`**: Pure declaration, no state.
- **`tensor_apply.cpp` / `.h`**: No static/global state. Functions operate on local data and call Python C API with the GIL (caller must hold it). The `PyObject_CallObject` and `PyTuple_SET_ITEM` calls operate on freshly created local objects.
- **`tensor_flatten.cpp` / `.h`**: Pure C++ tensor operations with no Python API usage and no global mutable state.
- **`tensor_list.cpp` / `.h`**: No static/global state. `recursive_to_list` creates new Python lists (owned). `tensor_to_list` properly acquires/releases GIL. `PyList_SET_ITEM` operates on a freshly created list.
- **`tensor_new.h`**: Pure declarations.
- **`tensor_dtypes.h`**, **`tensor_layouts.h`**, **`tensor_memoryformats.h`**: Pure declarations.

## Summary

The most actionable issue is the `static std::string sig` data race in `_validate_sparse_compressed_tensor_args_template` (Issue 3), which is a genuine concurrent write race on a static local variable. The `static PythonArgParser` instances (Issue 2) need verification against the `PythonArgParser` implementation. The global registries (Issues 1, 5) follow a safe write-once-read-many pattern protected by Python's import serialization.
