# Free-Threading Audit: toplevel__part4

**Files:** `Types.h`, `copy_utils.h`, `itt.cpp`, `itt.h`, `itt_wrapper.cpp`, `itt_wrapper.h`, `python_dimname.cpp`, `python_dimname.h`, `python_headers.h`, `serialization.cpp`, `serialization.h`, `shim_common.cpp`, `shim_conversion_utils.h`, `utils.cpp`, `utils.h`
**Overall Risk:** High

## Issues

### 1. Static `InternedStringsTable` accessed without synchronization
- **Category:** Static/global mutable state in Python-facing functions
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/python_dimname.cpp`
- **Line(s):** 24, 100-108
- **Description:** `kPyInternedStringToDimname` is a file-static mutable `InternedStringsTable` instance containing a `ska::flat_hash_map`. `THPDimname_parse` (line 80) calls both `lookup` and `addMapping` on this table without any locking. Under the GIL, concurrent calls were serialized, but under free-threading multiple threads can simultaneously read and write the hash map, causing data races, corrupted state, or crashes.
- **Suggested Fix:** Protect `kPyInternedStringToDimname` accesses with a `std::mutex`, or use a concurrent hash map. Alternatively, since the keys are interned Python strings (pointer-identity lookup), a `Py_BEGIN_CRITICAL_SECTION`-style lock could be used if all callers hold the GIL-replacement critical section, but an explicit C++ mutex is more straightforward.

### 2. Borrowed references from Python containers in `THPUtils_checkDimnameList`
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/python_dimname.cpp`
- **Line(s):** 71-77
- **Description:** `PyTuple_GET_ITEM` and `PyList_GET_ITEM` return borrowed references. Under free-threading, another thread could mutate the list (lists are mutable) between fetching the size and accessing the element, or the element could be decref'd concurrently, leaving a dangling pointer. Tuples are immutable so only the list path is affected.
- **Suggested Fix:** For the list path, use `Py_BEGIN_CRITICAL_SECTION(obj)` / `Py_END_CRITICAL_SECTION()` around the size check and element access, or use `PyList_GetItemRef` (Python 3.13+) to get a strong reference.

### 3. Borrowed references from Python containers in `THPUtils_unpackLongs` and related functions
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/utils.cpp`
- **Line(s):** 49-68, 71-81, 83-90
- **Description:** `THPUtils_unpackLongs`, `THPUtils_checkIntTuple`, and `THPUtils_unpackIntTuple` use `PyTuple_GET_ITEM` and `PyList_GET_ITEM` to get borrowed references from containers. For the list path, another thread could mutate the list concurrently, invalidating the borrowed reference before it is used. The tuple paths are safe (tuples are immutable).
- **Suggested Fix:** For list paths, acquire a critical section on the container or use `PyList_GetItemRef` to get an owned reference.

### 4. Borrowed references in pybind11 `type_caster` load methods
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/utils.cpp`
- **Line(s):** 374-397, 407-435
- **Description:** The `type_caster<at::IntArrayRef>::load` and `type_caster<at::SymIntArrayRef>::load` functions iterate over Python lists/tuples using `PyTuple_GET_ITEM` / `PyList_GET_ITEM` with borrowed references. For lists, concurrent mutation by another thread could cause use-after-free.
- **Suggested Fix:** Use a critical section on the list object or use `PyList_GetItemRef` for the list code path.

### 5. `PyGILState_Ensure`/`Release` in `torch::gdb::tensor_repr`
- **Category:** `PyGILState_Ensure`/`Release` incompatible with free-threading
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/utils.cpp`
- **Line(s):** 287, 325, 337
- **Description:** `tensor_repr` uses `PyGILState_Ensure` / `PyGILState_Release` which are not compatible with free-threading builds. In free-threaded Python, these APIs are stubs that do not provide synchronization.
- **Suggested Fix:** This function is only intended for gdb debugging, so the practical risk is low. However, for correctness under free-threading, it should use `PyThreadState_Ensure` or the recommended free-threading API for acquiring the thread state. Given it is a debug-only function, this could also be documented as unsupported under free-threading.

### 6. Static mutable `backCompatBroadcastWarn` and `backCompatKeepdimWarn` flags
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** High
- **File:** `torch/csrc/utils.cpp`
- **Line(s):** 181-198
- **Description:** `backCompatBroadcastWarn` and `backCompatKeepdimWarn` are file-static `bool` variables read/written by getter/setter functions that are exposed to Python. Under the GIL, these were safe; under free-threading they are technically data races. However, since they are simple bools used for backward-compat warnings, the practical impact of a torn read is negligible.
- **Suggested Fix:** Declare as `std::atomic<bool>` to eliminate the data race with minimal overhead.

### 7. `PyObject_HasAttrString` called in a loop in serialization
- **Category:** Python C API on shared objects without critical section
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/serialization.cpp`
- **Line(s):** 42-43
- **Description:** `doPartialRead<PyObject*>` calls `PyObject_HasAttrString(fildes, "readinto")` on each iteration. Under free-threading, if `fildes` is shared between threads, attribute access is not protected. In practice, the file object is unlikely to be shared, making this low risk. There is also a TODO comment acknowledging the repeated call.
- **Suggested Fix:** Cache the result of the attribute check outside the read loop, or use `PyObject_GetOptionalAttr` (3.13+) for a safer attribute lookup pattern.

## Summary

The highest-risk issue is the unprotected static `InternedStringsTable` in `python_dimname.cpp`, where concurrent `THPDimname_parse` calls will race on the hash map. Several functions also use borrowed references from Python lists without critical sections. The remaining files (`Types.h`, `copy_utils.h`, `itt_wrapper.cpp`, `itt.h`, `itt_wrapper.h`, `python_headers.h`, `serialization.h`, `shim_common.cpp`, `shim_conversion_utils.h`, `utils.h`) are either header-only type definitions, pure C++ code without Python mutable state, or AOT inductor infrastructure with no GIL-dependent patterns, and present no free-threading concerns.
