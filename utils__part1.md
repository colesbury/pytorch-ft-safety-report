# Free-Threading Audit: utils__part1

**Files:**
- `torch/csrc/utils/byte_order.cpp`
- `torch/csrc/utils/byte_order.h`
- `torch/csrc/utils/cpp_stacktraces.cpp`
- `torch/csrc/utils/cpp_stacktraces.h`
- `torch/csrc/utils/cuda_enabled.h`
- `torch/csrc/utils/device_lazy_init.cpp`
- `torch/csrc/utils/device_lazy_init.h`
- `torch/csrc/utils/disable_torch_function.cpp`
- `torch/csrc/utils/disable_torch_function.h`
- `torch/csrc/utils/generated_serialization_types.h`
- `torch/csrc/utils/init.cpp`
- `torch/csrc/utils/init.h`
- `torch/csrc/utils/invalid_arguments.cpp`
- `torch/csrc/utils/invalid_arguments.h`
- `torch/csrc/utils/nested.cpp`

**Overall Risk:** High

## Issues

### 1. Static global `PyObject*` for `disabled_torch_function` / `disabled_torch_dispatch`
- **Category:** Static/global mutable `PyObject*`
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/utils/disable_torch_function.cpp`
- **Line(s):** 10-11, 18-24, 26-32
- **Description:** `disabled_torch_function` and `disabled_torch_dispatch` are file-scope static `PyObject*` pointers that are read by `disabled_torch_function_impl()` / `disabled_torch_dispatch_impl()` and written by their `set_*` counterparts. Under the GIL these accesses were serialized. Without the GIL, concurrent calls to the getter and setter from different threads constitute a data race. The getter returns a bare (non-owning) pointer; if another thread overwrites the global between the read and use, the caller holds a dangling or stale pointer. Additionally, `has_torch_function_attr` on line 313 reads `disabled_torch_function` without any synchronization to compare against an attribute value.
- **Suggested Fix:** Use `std::atomic<PyObject*>` for both globals, or protect accesses with a lightweight lock. Since these are essentially set once during module init and then only read, an atomic with `memory_order_acquire`/`memory_order_release` would suffice. Alternatively, if these are truly set once, set them during module init under the import lock and document that they are immutable after init.

### 2. Global mutable `is_initialized` / `is_in_bad_fork` / `at_fork_registered` arrays without synchronization
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/utils/device_lazy_init.cpp`
- **Line(s):** 15-17, 22-25, 62, 65-67, 69-75, 80
- **Description:** The file-scope `std::array<bool, ...>` globals `is_initialized`, `is_in_bad_fork`, and `at_fork_registered` are read and written from multiple functions (`device_lazy_init`, `set_requires_device_init`, `is_device_in_bad_fork`, `set_device_in_bad_fork`, `register_fork_handler_for_device_init`). The code comment on line 29 says "Protected by the GIL." Under free-threading the GIL no longer provides this protection. Concurrent calls to `device_lazy_init` for the same device type could both see `is_initialized[i] == false`, leading to duplicate initialization. `set_requires_device_init` and `set_device_in_bad_fork` can race with readers.
- **Suggested Fix:** Replace each `bool` element with `std::atomic<bool>`, or protect the critical sections with a mutex. The `device_lazy_init` function already acquires the GIL (line 23); under free-threading, this `gil_scoped_acquire` does not serialize access, so an additional per-device-type `std::mutex` or `c10::once_flag` per device is needed.

### 3. `device_lazy_init` relies on `pybind11::gil_scoped_acquire` for mutual exclusion
- **Category:** `PyGILState_Ensure`/`Release` incompatible with free-threading
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/utils/device_lazy_init.cpp`
- **Line(s):** 28-29
- **Description:** `device_lazy_init` uses `pybind11::gil_scoped_acquire` and the comment explicitly states "Protected by the GIL." Under free-threading, acquiring the GIL is a no-op (or acquires a non-exclusive lock), so this no longer serializes access. The check-then-act pattern (`if (is_device_initialized(...)) return; ... is_initialized[...] = true;`) is a TOCTOU race.
- **Suggested Fix:** Introduce a per-device-type `std::mutex` (or a single mutex guarding the whole array) and hold it across the check-init-set sequence. Alternatively, use `c10::call_once` with per-device `c10::once_flag` entries, with the caveat noted in the existing code comment about ASAN.

### 4. `is_device_initialized` acquires GIL but the read is still racy
- **Category:** `PyGILState_Ensure`/`Release` incompatible with free-threading
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/utils/device_lazy_init.cpp`
- **Line(s):** 22-25
- **Description:** `is_device_initialized` acquires the GIL before reading `is_initialized[...]`. Under free-threading, this does not provide mutual exclusion against a concurrent writer in `set_requires_device_init` (line 66) which does NOT acquire the GIL. This is a plain data race on a non-atomic `bool`.
- **Suggested Fix:** Make `is_initialized` elements `std::atomic<bool>`, or protect all reads/writes with the same mutex.

### 5. `PyObject_HasAttrString` usage may suppress exceptions
- **Category:** Python C API on shared objects without `Py_BEGIN_CRITICAL_SECTION`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/utils/device_lazy_init.cpp`
- **Line(s):** 51
- **Description:** `PyObject_HasAttrString` silently swallows exceptions. Under free-threading this is more problematic because the attribute lookup on a shared object is not protected by a critical section. In CPython 3.14t, attribute access on shared objects should ideally use `Py_BEGIN_CRITICAL_SECTION` or the newer `PyObject_GetOptionalAttrString` API to avoid races on the object's dict.
- **Suggested Fix:** Replace with `PyObject_GetOptionalAttrString` (available since Python 3.13) which does not suppress exceptions, and wrap the call in a critical section if needed.

### 6. `has_torch_function_attr` uses borrowed reference from `PyObject_FastGetAttrString`
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/utils/disable_torch_function.cpp`
- **Line(s):** 311-315
- **Description:** `has_torch_function_attr` calls `PyObject_FastGetAttrString` and compares the returned pointer with the global `disabled_torch_function`. Under free-threading, the attribute could be modified concurrently on the object, and the global `disabled_torch_function` could be swapped. Both the attribute lookup and the global read need synchronization. The `PyObject_FastGetAttrString` likely returns an owned reference (via `py::object`), so the reference itself is safe, but the comparison with the unsynchronized global is racy.
- **Suggested Fix:** Load `disabled_torch_function` via an atomic read (see issue #1). The attribute lookup itself is likely fine if `PyObject_FastGetAttrString` properly handles free-threading internally.

### 7. `sequence_has_torch_function` uses `PySequence_Fast_GET_ITEM` (borrowed ref)
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/utils/disable_torch_function.cpp`
- **Line(s):** 328-337
- **Description:** `PySequence_Fast_GET_ITEM` returns a borrowed reference. Under free-threading, if another thread modifies the sequence (list) concurrently, the borrowed reference could become invalid before `check_has_torch_function` finishes using it. Tuples are immutable and safe, but lists are not.
- **Suggested Fix:** Use `PyList_GetItemRef` (Python 3.13+) to get a strong reference when the container is a list, or protect the entire iteration with `Py_BEGIN_CRITICAL_SECTION` on the sequence object.

### 8. `nested_tensor_ctor` uses `PyList_Size` and `PyList_GetItemRef` without critical section
- **Category:** Python C API on shared objects without `Py_BEGIN_CRITICAL_SECTION`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/utils/nested.cpp`
- **Line(s):** 45-73
- **Description:** `nested_tensor_ctor` calls `PyList_Size(data)` on line 45 and then iterates with `PyList_Size(data)` again on line 50, calling `PyList_GetItemRef(data, i)` in the loop. If `data` is mutated by another thread between the size check and the item access, the index could be out of bounds or the list contents could change. While `PyList_GetItemRef` is the correct strong-reference API, the size-then-iterate pattern is a TOCTOU issue.
- **Suggested Fix:** Wrap the entire loop (lines 49-73) in `Py_BEGIN_CRITICAL_SECTION(data)` / `Py_END_CRITICAL_SECTION()` to hold the list's per-object lock for the duration of iteration.

### 9. `format_invalid_args` uses `PyTuple_GET_ITEM` (borrowed ref) without critical section
- **Category:** Borrowed references from Python containers
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/utils/invalid_arguments.cpp`
- **Line(s):** 373-376
- **Description:** `PyTuple_GET_ITEM` returns a borrowed reference from the tuple. Tuples are immutable so this is generally safe, but the macro performs no error checking. The `PyDict_Next` iteration on the kwargs dict (lines 383-387) is already correctly wrapped in `Py_BEGIN_CRITICAL_SECTION`, which is good.
- **Suggested Fix:** No change strictly needed since tuples are immutable. For extra safety, consider using `PyTuple_GetItemRef` (Python 3.13+) to obtain a strong reference.

## Summary

The most critical issues are in `device_lazy_init.cpp` and `disable_torch_function.cpp`. The device lazy initialization code explicitly relied on the GIL for synchronization (per its own comments) and has multiple global mutable arrays (`is_initialized`, `is_in_bad_fork`, `at_fork_registered`) accessed without atomics or locks. The `disable_torch_function.cpp` file has two global `PyObject*` pointers read and written without synchronization. Several other borrowed-reference patterns in `disable_torch_function.cpp`, `nested.cpp`, and `invalid_arguments.cpp` also need attention for list/sequence iteration under free-threading.
