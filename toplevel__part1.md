# Free-Threading Audit: toplevel__part1

**Files:**
- `torch/csrc/CudaIPCTypes.cpp`
- `torch/csrc/CudaIPCTypes.h`
- `torch/csrc/DataLoader.cpp`
- `torch/csrc/DataLoader.h`
- `torch/csrc/Device.cpp`
- `torch/csrc/Device.h`
- `torch/csrc/DeviceAccelerator.cpp`
- `torch/csrc/DeviceAccelerator.h`
- `torch/csrc/Dtype.cpp`
- `torch/csrc/Dtype.h`
- `torch/csrc/DynamicTypes.cpp`
- `torch/csrc/DynamicTypes.h`
- `torch/csrc/Event.cpp`
- `torch/csrc/Event.h`
- `torch/csrc/Exceptions.cpp`

**Overall Risk:** High

## Issues

### 1. Static `worker_pids` map accessed without synchronization from Python methods
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/DataLoader.cpp`
- **Line(s):** 121, 129-172, 186-201, 213-219
- **Description:** The file-static `std::map<int64_t, std::set<pid_t>> worker_pids` is read and mutated by three different Python-callable functions (`THPModule_errorIfAnyWorkerFails`, `THPModule_setWorkerPIDs`, `THPModule_removeWorkerPIDs`) without any mutex protection. Under the GIL, these calls were serialized. Under free-threading, concurrent calls from different threads will race on the map, causing undefined behavior.
- **Suggested Fix:** Add a `std::mutex` guarding all accesses to `worker_pids`, or use `Py_BEGIN_CRITICAL_SECTION` on a shared lock object.

### 2. Global mutable `PyObject* THPUpperModuleOfDevice` without synchronization
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/Device.cpp`
- **Line(s):** 15, 58, 293
- **Description:** `THPUpperModuleOfDevice` is a global `PyObject*` set during `THPDevice_init` (line 293) and read during `THPDevice_pynew` (line 58). This is only a concern if `THPDevice_pynew` can be called before or concurrently with `THPDevice_init`, which is unlikely in practice since init runs during module import and the type is not usable before `PyType_Ready`. However, the pointer is written without any memory ordering guarantee that would make it visible to other threads.
- **Suggested Fix:** This is init-time-only state; the risk is very low. If desired, make it `std::atomic<PyObject*>` or ensure it is set before any other thread can access the type.

### 3. Global mutable `PyObject* THPEventClass` without synchronization
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/Event.cpp`
- **Line(s):** 14, 332-333
- **Description:** `THPEventClass` is set during `THPEvent_init` (line 333) and read in `THPEvent_Check` (Event.h line 21) and `THPEvent_elapsed_time` (line 219). Like `THPUpperModuleOfDevice`, this is init-time state, but the write has no ordering guarantee for concurrent readers.
- **Suggested Fix:** Low risk since it is set at module init. If desired, make it `std::atomic<PyObject*>` with relaxed/acquire-release ordering.

### 4. Global mutable `PyObject*` exception objects without synchronization
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/Exceptions.cpp`
- **Line(s):** 14-18, 23-138
- **Description:** Nine global `PyObject*` exception pointers (`THPException_FatalError`, `THPException_LinAlgError`, etc.) are written during `THPException_init` and read whenever exceptions are translated. Same init-time pattern as above.
- **Suggested Fix:** Low risk; init-time only. If desired, wrap with `std::atomic` or use `std::call_once`.

### 5. Lazy-init static `PyTypeObject*` cache in `getTypedStorageTypeObject`
- **Category:** Lazy init patterns (`static T* cache = nullptr; if (!cache) cache = ...`)
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/DynamicTypes.cpp`
- **Line(s):** 71-75
- **Description:** `getTypedStorageTypeObject()` uses a function-local `static PyTypeObject*` initialized by calling `loadTypedStorageTypeObject()`. In C++11+, static local initialization is thread-safe for the initialization itself, so two threads won't both call the initializer. However, `loadTypedStorageTypeObject()` calls `PyImport_ImportModule` and `PyObject_GetAttrString`, which require the GIL (or in free-threading, a valid thread state with critical sections). If the first call happens from a thread without proper Python state, it will crash. Additionally, `loadTypedStorageTypeObject` leaks references: it calls `PyImport_ImportModule` and `PyObject_GetAttrString` without decrementing references on the intermediate results (lines 61-62 and 64-68), and calls `PyObject_GetAttrString` twice for "TypedStorage" (lines 65 and 68), leaking the first result.
- **Suggested Fix:** The C++ static initialization handles the thread-safety of the lazy init itself. Ensure the function is first called while the caller has a valid Python thread state. Fix the reference leaks.

### 6. Global `dtype_registry` and `layout_registry` arrays without synchronization
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/DynamicTypes.cpp`
- **Line(s):** 13-18, 21-27, 29-43
- **Description:** `dtype_registry` and `layout_registry` are file-static `std::array` objects written by `registerDtypeObject` / `registerLayoutObject` and read by `getTHPDtype` / `getTHPLayout`. Under the GIL, registration (which happens at module init) was serialized with reads. Under free-threading, if any thread reads a dtype/layout while another thread is still registering dtypes (e.g., during concurrent module imports), there is a data race on the array elements.
- **Suggested Fix:** Registration happens at init time and reads happen later, so the practical risk is low if module init completes before use. For robustness, use `std::atomic<THPDtype*>` / `std::atomic<THPLayout*>` array elements, or protect with a `std::once_flag`.

### 7. `GetNewRefCountedSentData` accesses shared state outside mutex
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/CudaIPCTypes.cpp`
- **Line(s):** 218-251
- **Description:** `GetNewRefCountedSentData` acquires `ref_counters_mutex_` for the block at lines 219-237 to potentially allocate a new ref counters file, but then accesses `next_available_ref_counters_file_` (lines 238-249) outside the lock. Multiple concurrent callers could race on `set_counter`, `get_offset`, `counter_ptr`, `rotate_offset`, `have_offsets`, and the `reset()` call. This is a C++ concurrency bug that existed pre-free-threading, but would be exposed more readily without the GIL if called from Python code paths.
- **Suggested Fix:** Extend the mutex scope to cover all accesses to `next_available_ref_counters_file_` in this function, or restructure to do all work within the lock.

### 8. `PyTuple_GET_ITEM` returns borrowed references in `THPModule_setWorkerPIDs`
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/DataLoader.cpp`
- **Line(s):** 184, 188, 197
- **Description:** `PyTuple_GET_ITEM` returns a borrowed reference. Under free-threading, if another thread mutates or destroys the tuple concurrently, the borrowed reference could dangle. In practice, the tuple is a function argument that should remain alive for the duration of the call, but without `Py_BEGIN_CRITICAL_SECTION` on the tuple there is no formal guarantee.
- **Suggested Fix:** Use `Py_BEGIN_CRITICAL_SECTION(args)` / `Py_END_CRITICAL_SECTION()` around accesses to tuple items via `PyTuple_GET_ITEM`, or use `PyTuple_GetItem` which is safer under free-threading.

## Summary

The highest-risk issue is the unprotected static `worker_pids` map in `DataLoader.cpp`, which is directly mutated by multiple Python-callable functions with no locking. The `DynamicTypes.cpp` registries and `CudaIPCTypes.cpp` ref-counter access outside the mutex are medium-risk concurrency issues. The remaining global `PyObject*` pointers are low risk since they follow an init-once pattern during module setup.
