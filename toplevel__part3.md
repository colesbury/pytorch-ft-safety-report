# Free-Threading Audit: toplevel__part3

**Files:** `torch/csrc/QScheme.h`, `torch/csrc/Size.cpp`, `torch/csrc/Size.h`, `torch/csrc/Storage.cpp`, `torch/csrc/Storage.h`, `torch/csrc/StorageMethods.cpp`, `torch/csrc/StorageMethods.h`, `torch/csrc/StorageSharing.cpp`, `torch/csrc/StorageSharing.h`, `torch/csrc/Stream.cpp`, `torch/csrc/Stream.h`, `torch/csrc/THConcat.h`, `torch/csrc/THP.h`, `torch/csrc/TypeInfo.cpp`, `torch/csrc/TypeInfo.h`
**Overall Risk:** Medium

## Issues

### 1. Global mutable `THPStorageClass` pointer set during module post-init
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Storage.cpp`
- **Line(s):** 35, 486-489
- **Description:** `THPStorageClass` is a global `PyTypeObject*` initialized to `nullptr` and later set by `THPStorage_postInit`. Multiple readers (e.g. `THPStorage_Check` in `Storage.h` line 32, `THPStorage_CheckTypeExact` line 23) access this pointer without synchronization. Under GIL-free threading, if `THPStorage_postInit` races with any code path that calls `THPStorage_Check`, there is a torn-read risk. In practice this is mostly protected by module init ordering, but there is no formal guarantee.
- **Suggested Fix:** Use `std::atomic<PyTypeObject*>` for `THPStorageClass`, or ensure the write happens-before any concurrent reads via module init barriers.

### 2. Global mutable `THPStreamClass` pointer set during module init
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Stream.cpp`
- **Line(s):** 15, 538
- **Description:** `THPStreamClass` is a global `PyTypeObject*` set in `THPStream_init`. Readers like `THPStream_Check` (Stream.h line 22) and `THPStream_Wrap` (line 103) access it without synchronization. Same torn-read concern as `THPStorageClass` above.
- **Suggested Fix:** Use `std::atomic<PyTypeObject*>` or ensure a happens-before relationship from the write to all reads.

### 3. Static `std::vector<PyMethodDef>` in `THPStorage_init`
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/Storage.cpp`
- **Line(s):** 471
- **Description:** `THPStorage_init` uses a `static std::vector<PyMethodDef> methods` that is populated once. The `static` keyword means a second call would skip initialization, but the vector is mutated via `THPUtils_addPyMethodDefs` and its `.data()` pointer is assigned to `THPStorageType.tp_methods`. If module init were ever called concurrently (unlikely but not formally prevented), this would be a data race.
- **Suggested Fix:** No fix needed if module init is guaranteed single-threaded. Otherwise use `std::call_once`.

### 4. Static `decode_map` in `THPStorage_fromBuffer`
- **Category:** Lazy init patterns
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/StorageMethods.cpp`
- **Line(s):** 323-336
- **Description:** A `static const std::unordered_map` is initialized on first call. In C++11+, static local initialization is thread-safe by the standard, so this is safe. Noting for completeness only.
- **Suggested Fix:** None needed; C++ guarantees thread-safe static local initialization.

### 5. `PyBytes_AS_STRING` on borrowed references from tuple items
- **Category:** Borrowed references from Python containers
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/StorageSharing.cpp`
- **Line(s):** 173-174, 387, 483
- **Description:** `PyBytes_AS_STRING` is called on borrowed references obtained via `PyTuple_GET_ITEM` from argument tuples. Under free-threading, if the bytes object were concurrently modified or deallocated by another thread, the returned `char*` could be invalid. In practice, argument tuples are owned by the calling frame and not shared, so the risk is minimal.
- **Suggested Fix:** No immediate fix needed; argument tuples are effectively thread-local. If paranoia is warranted, incref the bytes objects before extracting their string pointers.

### 6. `THPStream.context` list accessed without critical section
- **Category:** Python C API on shared objects without `Py_BEGIN_CRITICAL_SECTION`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Stream.cpp`
- **Line(s):** 299-346, 364-404
- **Description:** `THPStream_enter` and `THPStream_exit` read and mutate `self->context` (a Python list) via `PyList_Append`, `PyList_GET_ITEM`, `PyList_SetSlice`, and `PyList_Size`. If two threads concurrently enter/exit the same stream context manager, these operations race on the list's internal state. Under the GIL this was serialized; under free-threading it is not.
- **Suggested Fix:** Wrap the context-list manipulation in `Py_BEGIN_CRITICAL_SECTION(self->context)` / `Py_END_CRITICAL_SECTION()`, or use a per-object mutex.

## Summary

The main concerns are the unsynchronized global `PyTypeObject*` pointers (`THPStorageClass`, `THPStreamClass`) set during module initialization and the unprotected mutation of the `THPStream.context` list in `__enter__`/`__exit__`. The global pointer issues are low practical risk if module init is single-threaded, but the stream context list manipulation is a real race if a stream object is shared across threads.
