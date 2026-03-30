# Free-Threading Audit: autograd__part3

**Files:**
- `torch/csrc/autograd/input_metadata.cpp`
- `torch/csrc/autograd/input_metadata.h`
- `torch/csrc/autograd/jit_decomp_interface.cpp`
- `torch/csrc/autograd/jit_decomp_interface.h`
- `torch/csrc/autograd/profiler.h`
- `torch/csrc/autograd/profiler_kineto.cpp`
- `torch/csrc/autograd/profiler_kineto.h`
- `torch/csrc/autograd/profiler_legacy.cpp`
- `torch/csrc/autograd/profiler_legacy.h`
- `torch/csrc/autograd/profiler_python.cpp`
- `torch/csrc/autograd/profiler_python.h`
- `torch/csrc/autograd/python_anomaly_mode.cpp`
- `torch/csrc/autograd/python_anomaly_mode.h`
- `torch/csrc/autograd/python_autograd.h`
- `torch/csrc/autograd/python_cpp_function.cpp`

**Overall Risk:** High

## Issues

### 1. Global mutable `JitDecompInterface*` pointer without synchronization
- **Category:** Static/global mutable state (Category 5)
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/jit_decomp_interface.cpp`
- **Line(s):** 6, 9-11, 13-15
- **Description:** The global `impl` pointer is set via `setJitDecompImpl` and read via `getJitDecompImpl` with no synchronization. Under free-threading, if `setJitDecompImpl` is called from one thread while another calls `getJitDecompImpl`, this is a data race. In practice this is set once during static initialization via `JitDecompRegisterer`, so the risk is low if initialization completes before any reads.
- **Suggested Fix:** Use `std::atomic<JitDecompInterface*>` for the global pointer, or document that it is only set during static initialization.

### 2. Lazy-init static `PyCodeObject*` caches in `getCode<>()` templates
- **Category:** Lazy init pattern (Category 6)
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/autograd/profiler_python.cpp`
- **Line(s):** 69-81, 84-96
- **Description:** `getCode<CallType::PyModuleCall>()` and `getCode<CallType::PyOptimizerCall>()` each use a `static auto code = []() { ... }()` pattern that acquires the GIL inside the lambda. While C++ guarantees thread-safe initialization of function-local statics, the stored value is a bare `PyCodeObject*` (borrowed reference from a Python attribute lookup). Under free-threading, the Python object could be collected or moved if no strong reference is held. The `res` pointer obtained from `.ptr()` is a borrowed reference that is never incref'd before being stored in the static.
- **Suggested Fix:** `Py_INCREF(res)` before storing it in the static, so the `PyCodeObject*` is a strong reference that prevents the object from being collected.

### 3. Static `py_gc_callback` global PyObject pointer without synchronization
- **Category:** Static/global mutable `PyObject*` (Category 1)
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/profiler_python.cpp`
- **Line(s):** 983, 1098-1099, 1125
- **Description:** `py_gc_callback` is a static global `PyObject*` that is set in `register_gc_callback` and cleared in `unregister_gc_callback`. Under free-threading, concurrent profiler start/stop from different threads could race on this pointer. The comment says "we are only registering on main thread while holding GIL" but the GIL no longer serializes access under free-threading.
- **Suggested Fix:** Use `std::atomic<PyObject*>` or protect access with a mutex.

### 4. `PyGILState_Ensure`/`Release` usage in profiler_python.cpp
- **Category:** `PyGILState_Ensure`/`Release` incompatible with free-threading (Category 4)
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/autograd/profiler_python.cpp`
- **Line(s):** 1075-1100, 1104-1133
- **Description:** `unregister_gc_callback()` and `PythonTracer::register_gc_callback()` use `PyGILState_Ensure`/`PyGILState_Release` which are not recommended for free-threaded Python. These should be replaced with the pybind11 `gil_scoped_acquire` RAII wrapper which handles both GIL and no-GIL builds.
- **Suggested Fix:** Replace `PyGILState_Ensure`/`PyGILState_Release` pairs with `pybind11::gil_scoped_acquire` (already used elsewhere in this file).

### 5. Global `cpp_function_types_map` and `cpp_function_types_set` without synchronization
- **Category:** Static/global mutable state in Python-facing functions (Category 5)
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/autograd/python_cpp_function.cpp`
- **Line(s):** 269-270, 300-306, 321-326, 328-339
- **Description:** `cpp_function_types_map` and `cpp_function_types_set` are static globals that are written by `registerCppFunction` and read by `functionToPyObject` and `THPCppFunction_Check`. Under the GIL these were safe because all Python-facing calls were serialized. Under free-threading, concurrent calls to `functionToPyObject` (which reads the map) and `registerCppFunction` (which writes to it) constitute a data race on `std::unordered_map` and `std::unordered_set`, which is undefined behavior.
- **Suggested Fix:** Protect with a `std::mutex`, or use a concurrent data structure. If registration only happens during module init, consider a read-only pattern after initialization is complete (e.g., `folly::Synchronized` or a reader-writer lock).

### 6. `functionToPyObject` TOCTOU race on `cdata->pyobj()`
- **Category:** Static/global mutable state in Python-facing functions (Category 5)
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/autograd/python_cpp_function.cpp`
- **Line(s):** 296-319
- **Description:** `functionToPyObject` first checks `cdata->pyobj()` is null (line 296), then if null creates a new Python wrapper and calls `cdata->set_pyobj(obj.release())` (line 315), and finally returns `cdata->pyobj()` (line 318). If the object already has a pyobj, it does `Py_INCREF(cdata->pyobj())` (line 297). Under free-threading, two threads could simultaneously call `functionToPyObject` on the same `cdata`, both see `pyobj()` as null, both create wrappers, and one overwrites the other's `set_pyobj`, leaking the first wrapper and potentially causing a use-after-free. The check-then-act on `pyobj()` is a classic TOCTOU race.
- **Suggested Fix:** Use a mutex or atomic compare-and-swap when setting `pyobj()`, or use `Py_BEGIN_CRITICAL_SECTION` on the node to serialize access.

### 7. `profiler_state_info_ptr` shared_ptr global without synchronization
- **Category:** Static/global mutable state (Category 5)
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/profiler_kineto.cpp`
- **Line(s):** 607, 847, 852, 856, 876
- **Description:** `profiler_state_info_ptr` is a `std::shared_ptr<ProfilerStateInfo>` at file scope that is written by `enableProfiler` (line 847), read by `isProfilerEnabledInMainThread` (line 852), read by `enableProfilerInChildThread` (line 856), and set to null by `disableProfiler` (line 876). `std::shared_ptr` assignment/read is not atomic by default. Under free-threading, a child thread calling `enableProfilerInChildThread` while the main thread calls `disableProfiler` (setting it to nullptr) is a data race.
- **Suggested Fix:** Use `std::atomic<std::shared_ptr<ProfilerStateInfo>>` (C++20) or protect with a mutex.

### 8. `memory_tracer` global unique_ptr without synchronization
- **Category:** Static/global mutable state (Category 5)
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/profiler_kineto.cpp`
- **Line(s):** 915-921
- **Description:** The global `memory_tracer` unique_ptr is lazily initialized in `startMemoryProfile` with a check-then-set pattern (`if (memory_tracer == nullptr)`). Under free-threading, concurrent calls would race.
- **Suggested Fix:** Use `std::once_flag`/`std::call_once` or a mutex.

### 9. Lazy-init static `prefixes` vector in `ValueCache::trimPrefixes`
- **Category:** Lazy init pattern (Category 6)
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/profiler_python.cpp`
- **Line(s):** 492-498
- **Description:** `trimPrefixes` uses a `static const auto prefixes = []() { ... }()` lambda that acquires the GIL and calls into Python. C++ guarantees thread-safe initialization of function-local statics, so the initialization itself is safe. However, the result is a `std::vector<std::string>` obtained from Python, and the GIL acquisition inside the initializer may interact poorly with free-threading if the Python interpreter state is not fully set up. This is a minor concern.
- **Suggested Fix:** No immediate action needed; the C++ static init guarantee handles the race. Verify the GIL acquisition path works under free-threading.

### 10. Borrowed reference from `PyDict_GetItemString` in `recordPyCall`
- **Category:** Borrowed references from Python containers (Category 3)
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/profiler_python.cpp`
- **Line(s):** 1210-1211
- **Description:** On Python < 3.13, `PyDict_GetItemString(locals, "self")` returns a borrowed reference. While there is a subsequent `Py_INCREF(self.get())`, under free-threading the borrowed reference could become dangling between the `PyDict_GetItemString` call and the `Py_INCREF`. The 3.13+ code path uses `PyMapping_GetItemString` which returns a new strong reference, which is correct.
- **Suggested Fix:** Use `PyDict_GetItemStringRef` (available since Python 3.13, backported via pythoncapi_compat) which returns a strong reference, or wrap the access in `Py_BEGIN_CRITICAL_SECTION`.

## Fixed Issues (not listed above — the original audit missed these)

The two most severe `profiler_python.cpp` issues were not identified by this audit but have been fixed:

- **`PyThreadState_Swap` into running threads' state** — the profiler iterated all threads and swapped into each one's `PyThreadState` to walk frames and install profile hooks. Under free-threading, those threads are actively running. Fixed by #178551 (replaced with `StopTheWorldGuard` + `PyThreadState_GetFrame` + `setprofileAllThreads`).

- **Shared `ValueCache` across threads** — a single `ValueCache` instance was passed by pointer to all `ThreadLocalResults`, meaning profiler callbacks on different threads wrote to the same maps concurrently. Fixed by #178552 (made `ValueCache` per-thread).

## Summary

The most significant free-threading issues are in `python_cpp_function.cpp` where global mutable containers (`cpp_function_types_map`/`cpp_function_types_set`) and the `functionToPyObject` TOCTOU race on `pyobj()` are exposed to data races. The profiler files have several global mutable state issues (`profiler_state_info_ptr`, `py_gc_callback`, `memory_tracer`) and use of `PyGILState_Ensure`/`Release` (which is fine for calling Python C APIs but should not be relied on as an implicit lock for C/C++ data). The profiler python code also stores bare borrowed `PyCodeObject*` pointers in statics without holding strong references.
