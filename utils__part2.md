# Free-Threading Safety Audit: torch/csrc/utils (Part 2)

## Files Reviewed
- `utils/nested.h`
- `utils/numpy_stub.h`
- `utils/object_ptr.cpp`
- `utils/object_ptr.h`
- `utils/out_types.cpp`
- `utils/out_types.h`
- `utils/pybind.cpp`
- `utils/pybind.h`
- `utils/pycfunction_helpers.h`
- `utils/pyobject_preservation.cpp`
- `utils/pyobject_preservation.h`
- `utils/python_arg_parser.cpp`
- `utils/python_arg_parser.h`
- `utils/python_compat.h`
- `utils/python_dispatch.cpp`

## Overall Risk: Medium

## Issues

### Issue 1: Static mutable `py::object` in `is_dtensor` (python_arg_parser.cpp:316-318)

**Severity: Low**
**Description:**
`is_dtensor()` contains a `static py::object base_td` that is lazily initialized on first call. While C++ "magic statics" guarantee the initialization itself is thread-safe, the `py::object` constructor calls `dtensor.attr("__torch_dispatch__").attr("__func__")` which performs Python attribute lookups. These attribute lookups are safe (they acquire the critical section internally under free-threading), but the static `py::object` itself permanently holds a reference to a Python object. The bigger concern is that `base_td` is compared via `.is()` against a freshly-fetched `sub_td` on every call -- both the static storage and the `.is()` comparison involve accessing the `py::object`'s internal `PyObject*` pointer. Under free-threading, the static `py::object`'s pointer read is safe because it is written exactly once (magic static), and `.is()` is just a pointer comparison. This pattern is acceptable.

However, the `#if IS_PYBIND_2_13_PLUS` / `#else` branch in `maybe_get_registered_torch_dispatch_rule()` (lines 280-295) has a potential issue in the `#else` branch: a `static const py::handle` is created from a leaked `py::object`. The `py::object` construction and `.release()` are not atomic, but magic statics guarantee single initialization. This is safe.

**No action required** -- these patterns are safe under free-threading due to C++ magic statics guaranteeing single initialization and subsequent read-only access.

### Issue 2: Global mutable `python_registrations_` (python_dispatch.cpp:39-42)

**Severity: Medium**
**Description:**
```cpp
static ska::flat_hash_map<
    c10::OperatorName,
    ska::flat_hash_map<c10::DispatchKey, std::shared_ptr<c10::SafePyObject>>>
    python_registrations_;
```
This file-scope static hash map is mutated in `initDispatchBindings` (via `python_registrations_[lib._resolve(name)].insert_or_assign(...)` at line 427) and read in `python_op_registration_trampoline_impl` (line 1080). Under the GIL, all Python-facing code was serialized, so concurrent reads/writes were impossible. Under free-threading, if two threads simultaneously register operator implementations or one registers while another dispatches, there is a data race on this map.

**Recommendation:** Protect `python_registrations_` with a mutex, or use a concurrent hash map. Reads in `python_op_registration_trampoline_impl` should also be synchronized.

### Issue 3: Global mutable `leaked_python_filenames_` (python_dispatch.cpp:34)

**Severity: Medium**
**Description:**
```cpp
static std::vector<std::string> leaked_python_filenames_;
```
This vector is appended to in `_dispatch_library` (line 515) and cleared in `_dispatch_clear_leaked_python_filenames` (line 537). Both operations are called from Python. Under the GIL, these were serialized. Under free-threading, concurrent calls to `_dispatch_library` or a concurrent clear while another thread appends would be a data race on the vector.

Additionally, after `emplace_back` at line 515, the code immediately takes `.back().c_str()` at line 516. If another thread calls `emplace_back` concurrently, the vector may reallocate, invalidating the pointer returned by `.back().c_str()` from the first thread.

**Recommendation:** Protect this vector with a mutex. Alternatively, store the filenames in a structure that does not invalidate pointers on insertion (e.g., `std::deque` or `std::list`), combined with a mutex.

### Issue 4: `PyGILState_Check` in `destroy_without_gil` (pybind.h:412)

**Severity: Low**
**Description:**
```cpp
if (Py_IsInitialized() && PyGILState_Check()) {
    pybind11::gil_scoped_release nogil;
    delete ptr;
}
```
`PyGILState_Check()` checks whether the current thread holds the GIL. Under free-threading (Python 3.13t+), the GIL does not exist, so `PyGILState_Check()` always returns 1 (the thread is always considered to "hold" the GIL). This means `pybind11::gil_scoped_release` will always be invoked, which is a no-op under free-threading. The behavior is functionally correct but the guard is meaningless under free-threading.

**Recommendation:** No immediate fix needed. The code is correct in practice because `gil_scoped_release` is a no-op under free-threading. Consider adding a comment noting this.

### Issue 5: Borrowed references from Python containers (python_arg_parser.h, python_arg_parser.cpp)

**Severity: Low**
**Description:**
Throughout `python_arg_parser.h` and `python_arg_parser.cpp`, borrowed references are obtained from Python tuples and lists using `PyTuple_GET_ITEM` and `PyList_GET_ITEM` (e.g., lines 426-429, 459-460, 502-503, 582, 667, 905, 934, 1055 in the .h and .cpp files). Under free-threading, if another thread mutates a list while this code iterates over it, the borrowed reference could become dangling.

However, in practice, these containers are the `args` and `kwargs` from the Python function call, which are local to the calling frame and not shared. The `args` tuple is immutable. Lists passed as arguments could theoretically be mutated by another thread, but this is a pre-existing unsafety that Python's own C API has in general -- and CPython's free-threading implementation addresses this at the C API level with per-object locks.

**Recommendation:** No immediate fix needed. CPython's free-threading implementation handles list/tuple access internally. For `PyList_GET_ITEM`, consider using `PyList_GetItemRef` (available in 3.13+) for critical paths if extra safety is desired.

### Issue 6: `PyDict_GetItem` without error checking (python_arg_parser.cpp:1727-1732)

**Severity: Low**
**Description:**
```cpp
obj = PyDict_GetItem(kwargs, param.python_name);
for (PyObject* numpy_name : param.numpy_python_names) {
    if (obj) break;
    obj = PyDict_GetItem(kwargs, numpy_name);
}
```
`PyDict_GetItem` returns a borrowed reference and suppresses errors. Under free-threading, `PyDict_GetItem` is deprecated in favor of `PyDict_GetItemRef` (which returns a strong reference). However, the comment at line 1725 notes these are kwargs local to the current function call, so concurrent mutation is unlikely.

**Recommendation:** Consider migrating to `PyDict_GetItemRef` for robustness under free-threading, though the risk is low since kwargs are function-local.

## Safe Patterns Identified

1. **`type_map` and `numpy_compatibility_arg_names`** (python_arg_parser.cpp:27, 75): File-scope statics with constant initialization, read-only after program start. Safe.

2. **`should_allow_numbers_as_tensors`** (python_arg_parser.cpp:92): Uses a function-local `static std::unordered_set` with constant initialization. Safe via C++ magic statics (thread-safe initialization, read-only after).

3. **`parseKind` and `parseAliasAnalysisKind`** (python_dispatch.cpp:44-63): Function-local static maps with constant initialization. Safe via C++ magic statics.

4. **`maybe_get_registered_torch_dispatch_rule`** (python_arg_parser.cpp:280-295): Uses `PYBIND11_CONSTINIT static py::gil_safe_call_once_and_store` on pybind11 2.13+. This is explicitly designed for thread-safe lazy initialization. The fallback path also uses a magic static. Safe.

5. **`THPPointer` / `THPObjectPtr`** (object_ptr.h/cpp): RAII pointer wrapper. Not shared across threads by design; destructor calls `Py_DECREF` which is atomic under free-threading. Safe when used as intended (single-owner).

6. **`PyObjectPreservation`** (pyobject_preservation.cpp/h): Already uses `compare_exchange_strong`, `compare_exchange_weak`, and explicit memory ordering. Clearly designed for thread safety. Safe.

7. **`out_types.cpp`**: Pure function, no global/static state. Safe.

8. **Header-only files** (`nested.h`, `numpy_stub.h`, `python_compat.h`, `pycfunction_helpers.h`): Declarations and inline helpers only, no mutable shared state. Safe.

9. **`pybind.cpp`**: Type casters are stateless conversion functions. Safe.
