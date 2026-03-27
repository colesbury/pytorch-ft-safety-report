# Free-Threading Audit: cuda__part1

**Files:**
- `torch/csrc/cuda/CUDAPluggableAllocator.cpp`
- `torch/csrc/cuda/CUDAPluggableAllocator.h`
- `torch/csrc/cuda/Event.cpp`
- `torch/csrc/cuda/Event.h`
- `torch/csrc/cuda/GdsFile.cpp`
- `torch/csrc/cuda/GdsFile.h`
- `torch/csrc/cuda/Graph.cpp`
- `torch/csrc/cuda/GreenContext.cpp`
- `torch/csrc/cuda/MemPool.cpp`
- `torch/csrc/cuda/Module.cpp`
- `torch/csrc/cuda/Module.h`
- `torch/csrc/cuda/Stream.cpp`
- `torch/csrc/cuda/Stream.h`
- `torch/csrc/cuda/THCP.h`
- `torch/csrc/cuda/comm.cpp`

**Overall Risk:** High

## Issues

### 1. Global mutable `current_custom_allocator` shared_ptr without synchronization
- **Category:** Static/global mutable state in Python-facing functions
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/cuda/CUDAPluggableAllocator.cpp`
- **Line(s):** 365, 369, 389-390, 394
- **Description:** The namespace-scope `std::shared_ptr<CUDAAllocator> current_custom_allocator` is written by `changeCurrentAllocator` (line 390) and read by `getCurrentAllocator` (line 369) and `custom_raw_deleter` (line 394). `custom_raw_deleter` is the deleter registered with every allocation from the pluggable allocator and may be called from any thread at any time. Under free-threading, concurrent reads/writes of a `shared_ptr` are a data race (undefined behavior) since `shared_ptr` copy/assignment is not atomic at the object level. A thread calling `custom_raw_deleter` while another calls `changeCurrentAllocator` could crash or corrupt the shared_ptr's control block.
- **Suggested Fix:** Use `std::atomic<std::shared_ptr<...>>` (C++20) or protect all accesses with a mutex, or use `std::atomic_load`/`std::atomic_store` on the `shared_ptr`.

### 2. Global mutable `device_count` without synchronization
- **Category:** Static/global mutable state
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/cuda/CUDAPluggableAllocator.cpp`
- **Line(s):** 10, 379
- **Description:** The namespace-scope `int device_count = 0` is read by `createCustomAllocator` (line 379) but is never written after initialization (it is zero-initialized at file scope and never assigned again in this file). If it is intended to be set elsewhere, it would need synchronization. As it stands, it is always 0 when passed to `allocator->init()`, which is likely a latent bug but not a threading issue per se.
- **Suggested Fix:** Investigate whether `device_count` should be initialized properly. If it is meant to be written, add synchronization.

### 3. Global mutable `PyObject* THCPEventClass` without synchronization
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/cuda/Event.cpp`
- **Line(s):** 16, 268
- **Description:** `THCPEventClass` is a global `PyObject*` set during `THCPEvent_init` (line 268) and read in `THCPEvent_Check` (Event.h line 15-17). This is init-time-only state: it is written once during module initialization and read subsequently. Under free-threading, if `THCPEvent_Check` is called before module init completes, the read is a data race. In practice this is unlikely since the type is not usable before init, but the write has no memory ordering guarantee.
- **Suggested Fix:** Low risk. If desired, make it `std::atomic<PyObject*>` with acquire/release semantics.

### 4. Static `PythonArgParser` in `THCPEvent_from_ipc_handle` is not thread-safe
- **Category:** Lazy init patterns / static mutable C++ state
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/cuda/Event.cpp`
- **Line(s):** 68
- **Description:** `THCPEvent_from_ipc_handle` contains a `static torch::PythonArgParser parser(...)`. The `PythonArgParser::parse()` method mutates internal state (it populates parsed arguments into internal buffers). Under free-threading, if two threads call `from_ipc_handle` concurrently, they will both use the same static `parser` object, racing on its internal mutable state.
- **Suggested Fix:** Make the `PythonArgParser` a local variable (not `static`), or protect parsing with a mutex. The `static` is likely a performance optimization to avoid re-parsing format strings; the format-string parsing cost is typically negligible compared to the IPC handle operation.

### 5. `PyGILState_Ensure` / `PyGILState_Release` in cuda mutex lock/unlock
- **Category:** PyGILState_Ensure/Release
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/cuda/Module.cpp`
- **Line(s):** 495, 513, 519
- **Description:** `THCPModule_cudaLockMutex` calls `PyGILState_Ensure()` and stores the result in the file-static `cudaMutexGILState`, and `THCPModule_cudaUnlockMutex` calls `PyGILState_Release(cudaMutexGILState)`. The `PyGILState_Ensure`/`Release` API is deprecated and does not work correctly under free-threading (there is no GIL to ensure). Additionally, `cudaMutexGILState` is a single global variable written by `cudaLockMutex` and read by `cudaUnlockMutex` without any synchronization -- if two threads call these concurrently, they race on the variable. The entire pattern of holding the GIL while holding the CUDA free mutex is a GIL-era serialization mechanism that is incompatible with free-threading.
- **Suggested Fix:** Under free-threading, remove the `PyGILState_Ensure`/`PyGILState_Release` calls. The CUDA free mutex itself provides the necessary serialization for the CUDA operations; the GIL manipulation was only needed because other Python threads might try to acquire the CUDA mutex while holding the GIL, risking deadlock. Under free-threading, the deadlock risk from GIL ordering is eliminated. Consider using `Py_BEGIN_CRITICAL_SECTION` if Python object protection is needed.

### 6. Global mutable `PyObject* THCPStreamClass` without synchronization
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/cuda/Stream.cpp`
- **Line(s):** 14, 203
- **Description:** `THCPStreamClass` is set during `THCPStream_init` (line 203) and read in `THCPStream_Check` (Stream.h line 16-18). Same init-time-only pattern as `THCPEventClass`.
- **Suggested Fix:** Low risk. If desired, make it `std::atomic<PyObject*>`.

### 7. `THCPModule_attachOutOfMemoryObserver` captures PyObject without thread-safe reference management
- **Category:** Python C API without `Py_BEGIN_CRITICAL_SECTION`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/cuda/Module.cpp`
- **Line(s):** 996-1018
- **Description:** The function increfs `observer` (line 1000) and captures it in a lambda that calls `py::gil_scoped_acquire` before invoking the Python callback (line 1006). Under free-threading, `py::gil_scoped_acquire` is a no-op for the GIL but still ensures a valid thread state. The captured raw `PyObject*` pointer is safe as long as the reference is held. However, if the observer lambda is invoked from a C++ allocator thread that does not have a Python thread state, `py::gil_scoped_acquire` will create one, which is correct. The `Py_XINCREF` on line 1000 should use `Py_NewRef` or be done under a critical section for safety, but since this is called from Python with the caller holding a reference, the refcount operation is safe in practice.
- **Suggested Fix:** Low practical risk. Consider using `py::object` to hold the reference (RAII) rather than manual `Py_XINCREF`, which would also handle cleanup if the observer is ever removed.

### 8. Borrowed references from `PyTuple_GetItem` stored in `THPObjectPtr`
- **Category:** Borrowed refs from Python containers
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/cuda/Module.cpp`
- **Line(s):** 745-756
- **Description:** In `THCPModule_memorySnapshot`, `PyTuple_GetItem` returns a borrowed reference, but the results are stored in `THPObjectPtr` (lines 745-756), which will call `Py_XDECREF` on destruction. This is a reference counting bug regardless of free-threading: the borrowed reference will be double-decremented. Under free-threading, this can cause use-after-free if the refcount reaches zero in one thread while another is using the object. Note: `THPObjectPtr` calls `Py_XDECREF` in its destructor, so wrapping a borrowed reference in `THPObjectPtr` is incorrect.
- **Suggested Fix:** Use `PyTuple_GetItem` and store the result as a raw `PyObject*`, or use `PyTuple_GET_ITEM` (which also returns a borrowed reference) and do not wrap it in `THPObjectPtr`. Alternatively, use `Py_NewRef(PyTuple_GetItem(...))` if wrapping in `THPObjectPtr`.

### 9. `THCPModule_cudaJiteratorCompileAndLaunchKernel` uses `PyTuple_GET_ITEM` / `PyDict_Next` on containers
- **Category:** Borrowed refs from Python containers
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/cuda/Module.cpp`
- **Line(s):** 367-387
- **Description:** The function uses `PyTuple_GET_ITEM` to extract tensors from a tuple (line 371) and `PyDict_Next` to iterate over kwargs (line 385). The code correctly wraps the `PyDict_Next` call in `Py_BEGIN_CRITICAL_SECTION(kwargs_o)` / `Py_END_CRITICAL_SECTION()` (lines 384, 388), which is good practice for free-threading. The `PyTuple_GET_ITEM` calls on the tensors tuple are safe because tuples are immutable and the borrowed references are used immediately. This is already handled correctly.
- **Suggested Fix:** None needed; the critical section usage is correct.

### 10. `THCPModule_initExtension` sets module attributes without critical section
- **Category:** Python C API without `Py_BEGIN_CRITICAL_SECTION`
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/cuda/Module.cpp`
- **Line(s):** 1579-1616
- **Description:** `THCPModule_initExtension` imports `torch.cuda`, creates a tuple of default generators, and sets module attributes via `PyObject_SetAttrString`. Under free-threading, if two threads try to initialize CUDA concurrently (despite the `lazyInitDevice` guard), the `PyObject_SetAttrString` calls could race. However, CUDA initialization is typically guarded by `device_lazy_init` which uses `std::once` semantics, making concurrent init unlikely.
- **Suggested Fix:** Low risk due to the init guard. The `PyObject_SetAttrString` API is internally thread-safe in CPython 3.14t for dict operations on module dicts, so no additional protection is needed.
