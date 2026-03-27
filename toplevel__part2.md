# Free-Threading Audit: toplevel__part2

**Files:**
- `torch/csrc/Exceptions.h`
- `torch/csrc/Export.h`
- `torch/csrc/Generator.cpp`
- `torch/csrc/Generator.h`
- `torch/csrc/Layout.cpp`
- `torch/csrc/Layout.h`
- `torch/csrc/MemoryFormat.cpp`
- `torch/csrc/MemoryFormat.h`
- `torch/csrc/Module.cpp`
- `torch/csrc/Module.h`
- `torch/csrc/PyInterpreter.cpp`
- `torch/csrc/PyInterpreter.h`
- `torch/csrc/PyInterpreterHooks.cpp`
- `torch/csrc/PyInterpreterHooks.h`
- `torch/csrc/QScheme.cpp`

**Overall Risk:** High

## Issues

### 1. Global mutable `THPGeneratorClass` PyObject pointer
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Generator.cpp`
- **Line(s):** 21
- **Description:** `PyObject* THPGeneratorClass = nullptr` is a global mutable pointer set during module init (`THPGenerator_init`, line 357) and read by `THPGenerator_Wrap` (line 387), `THPGenerator_NewWithVar` (line 403), `THPGenerator_initDefaultGenerator` (line 24), and the `THPGenerator_Check` macro. Under free-threading, if multiple sub-interpreters or concurrent `import torch` init sequences occur, this is a data race. In practice, module init is typically single-threaded, so the risk is moderate.
- **Suggested Fix:** This is set once during module init and then read-only. If multi-interpreter support is needed, wrap in `std::atomic<PyObject*>` or use a `once_flag`. For the current single-interpreter case, the existing pattern is likely safe since Python serializes module init, but marking it `std::atomic` would be defensive.

### 2. Global mutable exception `PyObject*` pointers in `Exceptions.h`
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Exceptions.h`
- **Line(s):** 148-152
- **Description:** `THPException_FatalError`, `THPException_LinAlgError`, `THPException_OutOfMemoryError`, `THPException_DistError`, `THPException_DistBackendError`, `THPException_DistNetworkError`, `THPException_DistStoreError`, `THPException_DistQueueEmptyError`, and `THPException_AcceleratorError` are global `PyObject*` pointers. They are set during `THPException_init` (called from `initModule`) and then read from exception-handling macros (`CATCH_CORE_ERRORS`). Same write-once-then-read-only pattern as above.
- **Suggested Fix:** Since these are initialized once during module init and never modified afterward, they are likely safe in practice. For correctness under free-threading, they could be declared `std::atomic<PyObject*>` or use `Py_MOD_PER_INTERPRETER_GIL_NOT_USED`-compatible patterns.

### 3. `LogAPIUsageOnceFromPython` uses unprotected static `std::unordered_set`
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/Module.cpp`
- **Line(s):** 2125-2131
- **Description:** `LogAPIUsageOnceFromPython` contains a `static std::unordered_set<std::string> seen` that is read and written without synchronization. The comment says "Guaranteed to be invoked from Python under GIL, no locking on map needed" -- this guarantee no longer holds under free-threading. Concurrent calls from multiple Python threads will race on the `seen.count()` / `seen.insert()` operations, causing undefined behavior (corrupted hash table).
- **Suggested Fix:** Protect with a `std::mutex`, or use a concurrent set. Alternatively, use `std::call_once` per event string, though that's impractical for dynamic keys. A simple `static std::mutex` guarding the set is the most straightforward fix.

### 4. `THPModule_initNames` uses unprotected static `std::vector<std::string>`
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Module.cpp`
- **Line(s):** 162-189
- **Description:** `THPModule_initNames` contains a `static std::vector<std::string> names` that stores type name strings whose `.c_str()` pointers are assigned to `tp_name` fields. Under free-threading, concurrent calls would race on `names.reserve()`, `names.emplace_back()`, and the underlying reallocations could invalidate previously returned `.c_str()` pointers, causing use-after-free in other threads reading `tp_name`. In practice this is called early during init, so risk is moderate.
- **Suggested Fix:** Protect with a `std::mutex`, or assert single-threaded init context.

### 5. `THPModule_addDocStr` uses unprotected static `std::vector<std::string>`
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Module.cpp`
- **Line(s):** 433-491
- **Description:** `THPModule_addDocStr` contains a `static std::vector<std::string> all_docs` that stores docstrings whose `.c_str()` pointers are assigned to function/method/type `ml_doc`/`tp_doc` fields. A vector reallocation from a concurrent push_back would invalidate all previously returned `.c_str()` pointers, causing dangling pointers. This is likely called during import which is typically serial, but under free-threading concurrent module loading is possible.
- **Suggested Fix:** Protect with a `std::mutex`, or use `std::deque` (which doesn't invalidate pointers on push_back) plus a mutex.

### 6. Static `methods` vector populated and used during module init
- **Category:** Module init setting global state concurrently
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/Module.cpp`
- **Line(s):** 2121, 2226-2248
- **Description:** `static std::vector<PyMethodDef> methods` is populated during `initModule()` by multiple `THPUtils_addPyMethodDefs` calls. If `initModule` were called concurrently (e.g., from multiple sub-interpreters), this would be a data race. Python's import machinery typically serializes this, but the static variable is shared across all interpreters.
- **Suggested Fix:** Use `std::call_once` or move to local scope if feasible. The `PyUnstable_Module_SetGIL(module, Py_MOD_GIL_NOT_USED)` on line 2255 signals free-threading readiness, so this init code should be made safe.

### 7. Static `module` pointer in Module.cpp
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/Module.cpp`
- **Line(s):** 155
- **Description:** `static PyObject* module` is set during `initModule()` (line 2252). It is file-scoped and used within the same function. Since Python extension module init (`PyInit_*`) is called once per interpreter and the import lock serializes it, this is low risk. However, under free-threading with sub-interpreters, this static could be a concern.
- **Suggested Fix:** Make this a local variable within `initModule()` rather than file-static, since it is only used there (already aliased as `py_module` on line 2398).

### 8. `THPDefaultCPUGenerator` global pointer
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/Module.cpp`
- **Line(s):** 157, 3000-3006
- **Description:** `static THPGenerator* THPDefaultCPUGenerator` is set once during `initModule()` and exported as `default_generator` attribute. Write-once-then-read pattern, low risk.
- **Suggested Fix:** No change needed if init is guaranteed single-threaded. Otherwise `std::atomic`.

### 9. `THPGenerator_Wrap` TOCTOU race on pyobj check
- **Category:** Python C API on shared objects without critical section
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Generator.cpp`
- **Line(s):** 376-388
- **Description:** `THPGenerator_Wrap` checks `pyobj(gen)` to see if a Python object is already associated with the generator. If non-null, it does `Py_INCREF(obj)` and returns it. Under free-threading, another thread could concurrently clear or replace the pyobj between the check and the incref, leading to a use-after-free or dangling reference. This is a classic TOCTOU pattern. Additionally, the pyobj returned is a borrowed reference that could become invalid.
- **Suggested Fix:** Use `Py_BEGIN_CRITICAL_SECTION` / `Py_END_CRITICAL_SECTION` around the check-and-incref, or use `PyUnstable_TryIncRef` on the loaded pyobj.

### 10. `THPGenerator_pickleSetState` uses borrowed `PyTuple_GET_ITEM` references
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/Generator.cpp`
- **Line(s):** 269-279
- **Description:** `THPGenerator_pickleSetState` calls `PyTuple_GET_ITEM(state, 0)`, `PyTuple_GET_ITEM(state, 1)`, and `PyTuple_GET_ITEM(state, 2)` to get borrowed references, then passes them to `THPGenerator_manualSeed`, `THPGenerator_setOffset`, and `THPGenerator_setState`. Under free-threading, if another thread mutates the tuple concurrently, these borrowed references could become dangling. In practice, `__setstate__` tuples are typically not shared, so the risk is moderate.
- **Suggested Fix:** Use `PyTuple_GetItem` (which does error checking) or hold the tuple with `Py_BEGIN_CRITICAL_SECTION`, or use `Py_NewRef(PyTuple_GET_ITEM(...))` to get owned references.

### 11. `pytorch_duplicate_guard` uses unprotected static int
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** High
- **File:** `torch/csrc/Module.cpp`
- **Line(s):** 3051-3058
- **Description:** `pytorch_duplicate_guard` uses a `static int initialized` flag without synchronization. Under free-threading, concurrent initialization could race. However, this is guarded by `call_duplicate_guard` being a file-scope static object, which is initialized before `main()`.
- **Suggested Fix:** Use `std::atomic<int>` or `std::once_flag`.

### 12. `disabled_torch_function` and `disabled_torch_dispatch` global PyObject pointers
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/utils/disable_torch_function.cpp` (set from `torch/csrc/Module.cpp` lines 3032-3037)
- **Line(s):** Module.cpp:3032-3037, disable_torch_function.cpp:10-11
- **Description:** `torch::set_disabled_torch_function_impl` and `torch::set_disabled_torch_dispatch_impl` write to global `static PyObject*` pointers without synchronization. These are read frequently from `disabled_torch_function_impl()` / `disabled_torch_dispatch_impl()` in hot paths. While set once during module init, the reads are unsynchronized under free-threading.
- **Suggested Fix:** Use `std::atomic<PyObject*>` with relaxed or acquire/release ordering, since these follow a write-once-then-read pattern.

## Summary

The highest-risk issue is the `LogAPIUsageOnceFromPython` function (Issue 3) which explicitly relied on GIL protection (per its comment) for a static `unordered_set` that will now be racily accessed. The `THPModule_initNames` and `THPModule_addDocStr` functions (Issues 4-5) have similar unprotected static containers whose pointer-invalidating reallocations could cause use-after-free. Most other issues follow the write-once-during-init-then-read pattern and are lower risk in practice, though they should ideally use atomic types for correctness under the free-threading memory model.
