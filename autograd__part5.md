# Free-Threading Audit: autograd__part5

**Files:**
- `torch/csrc/autograd/python_saved_variable_hooks.cpp`
- `torch/csrc/autograd/python_saved_variable_hooks.h`
- `torch/csrc/autograd/python_sparse_functions.h`
- `torch/csrc/autograd/python_special_functions.h`
- `torch/csrc/autograd/python_torch_functions.h`
- `torch/csrc/autograd/python_torch_functions_manual.cpp`
- `torch/csrc/autograd/python_variable.cpp`
- `torch/csrc/autograd/python_variable.h`
- `torch/csrc/autograd/python_variable_indexing.cpp`
- `torch/csrc/autograd/python_variable_indexing.h`
- `torch/csrc/autograd/record_function_ops.cpp`
- `torch/csrc/autograd/record_function_ops.h`
- `torch/csrc/autograd/saved_variable.cpp`
- `torch/csrc/autograd/saved_variable.h`
- `torch/csrc/autograd/saved_variable_hooks.h`

**Overall Risk:** High

## Issues

### 1. Global mutable `THPVariableClass` and `ParameterClass` PyObject pointers
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_variable.cpp`
- **Line(s):** 322-324
- **Description:** `THPVariableClass` and `ParameterClass` are global mutable `PyObject*` pointers declared with `TORCH_PYTHON_API extern` in `python_variable.h` (lines 37-38) and assigned in `python_variable.cpp`. They are read pervasively (e.g., `THPVariable_Check`, `THPVariable_CheckTypeExact`, `THPVariable_WrapWithType`). Under the GIL these were safe because they are set during module init and only read afterwards. Under free-threading the init-time writes and subsequent reads are still likely safe because module init is sequenced before use, but there is no formal publication barrier guaranteeing visibility to other threads. If any code path reads these before module init completes on another thread, it would see stale values.
- **Suggested Fix:** Declare as `inline std::atomic<PyObject*>` or ensure they are set before any thread can access them via appropriate barriers. In practice, Python module init should provide the ordering, so this is lower priority.

### 2. Global mutable `THPVariableFunctionsModule` PyObject pointer
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_torch_functions_manual.cpp`
- **Line(s):** 47
- **Description:** `THPVariableFunctionsModule` is a global `PyObject*` set during `initTorchFunctions()` (line 597) and read from many `HANDLE_TH_ERRORS` functions. Same init-then-read pattern as above. Declared `extern` in the header at `python_torch_functions.h:5`.
- **Suggested Fix:** Same as issue 1 -- ensure publication via atomic or rely on module init ordering.

### 3. Global mutable `device_to_py_class_` array
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_variable.cpp`
- **Line(s):** 344-345, 347-360, 362-364
- **Description:** `device_to_py_class_` is a static array of `PyObject*` pointers written by `registerPythonTensorClass()` and read by `getPythonTensorClass()`. Under free-threading, concurrent calls to `registerPythonTensorClass` from different threads (or a register racing with reads in `THPVariable_WrapWithType`) would be a data race.
- **Suggested Fix:** Protect with a mutex, or use `std::atomic<PyObject*>` for each array element.

### 4. Lazy-init caching pattern in `DEFINE_CACHING_PYTHON_IMPORT_GETTER` (pre-pybind 2.13)
- **Category:** Lazy init patterns
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/autograd/python_variable.cpp`
- **Line(s):** 839-844
- **Description:** When `IS_PYBIND_2_13_PLUS` is *not* defined, the `DEFINE_CACHING_PYTHON_IMPORT_GETTER` macro uses a plain `static py::handle` with no synchronization. If two threads call one of these getters concurrently (e.g., `get_dtensor_class_impl()`), they may both execute the initialization simultaneously, causing a data race on the static `storage` variable and potentially double-releasing the Python object. There are approximately 10 uses of this macro (lines 846-898).
- **Suggested Fix:** This is already fixed in the `IS_PYBIND_2_13_PLUS` path which uses `py::gil_safe_call_once_and_store`. Ensure that the pybind11 version used with free-threading builds is always 2.13+, or backport the thread-safe pattern to the else branch.

### 5. Global `dtensor_interned_strings` written during init without synchronization
- **Category:** Module init setting global state concurrently
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_variable.cpp`
- **Line(s):** 985, 988-999
- **Description:** `dtensor_interned_strings` is a static struct of `PyObject*` pointers filled by `intern_dtensor_strings()` during module init. These are then read by all the DTensor dispatch fast-path code. Under free-threading, if module init is not fully serialized before use, reads could see partially initialized values.
- **Suggested Fix:** Ensure module init completes before any DTensor dispatch can occur (likely guaranteed by Python import mechanics). Otherwise, use `std::atomic` or `std::call_once`.

### 6. `PyDict_GetItemWithError` returns borrowed reference
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_variable.cpp`
- **Line(s):** 1501-1503, 2223-2229, 1862
- **Description:** Multiple calls to `PyDict_GetItemWithError` return borrowed references. Under free-threading, another thread could modify the dict concurrently, causing the borrowed reference to become dangling before it is used. For example at line 1502, `custom_op_handler` is a borrowed ref used immediately at line 1504 in `checked_vectorcall`. Similarly at lines 2223 and 2228, `runtime_schema_info` is borrowed and then used. At line 1862, `item` from `PyDict_GetItemWithError` on `kwargs_schema` is borrowed and then stored in a tuple.
- **Suggested Fix:** Use `PyDict_GetItemRef` (Python 3.13+) which returns a strong reference, or use `Py_BEGIN_CRITICAL_SECTION` / `Py_END_CRITICAL_SECTION` around the dict access and use of the borrowed reference.

### 7. `static PythonArgParser` in multiple functions
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/autograd/python_torch_functions_manual.cpp`, `torch/csrc/autograd/python_variable.cpp`
- **Line(s):** 75, 123, 250, 274, 299, 332, 432, 467, 503, 605, 629, 758, 1729 (and many more)
- **Description:** `static PythonArgParser parser(...)` appears in many functions. `PythonArgParser` performs lazy initialization internally (building the argument descriptors). If two threads enter the same function concurrently for the first time, the lazy init of the static local could race. However, C++11 guarantees thread-safe initialization of function-local statics via `static` initialization, so the construction itself is safe. The concern is whether `PythonArgParser::parse()` mutates shared state. This would need to be checked in the `PythonArgParser` implementation.
- **Suggested Fix:** Verify that `PythonArgParser::parse()` does not mutate the static parser object, or if it does, protect it accordingly.

### 8. `static std::vector<PyMethodDef> torch_functions` in `initTorchFunctions`
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/autograd/python_torch_functions_manual.cpp`
- **Line(s):** 579
- **Description:** `static std::vector<PyMethodDef> torch_functions` is populated during `initTorchFunctions`. Since this is called during module init which should be serialized, the risk is low. But if `initTorchFunctions` were ever called concurrently, this would be a data race.
- **Suggested Fix:** No action needed if init ordering is guaranteed.

### 9. `static std::mutex` and `static FastMap` for sharding propagator cache cleanup
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/autograd/python_variable.cpp`
- **Line(s):** 1269-1274
- **Description:** `native_sharding_propagator_cache_cleanup_mutex` and `all_thread_caches` are properly protected by a mutex, so the thread-local cache registration and cleanup are already thread-safe. The `NativeShardingPropagatorCache` itself is thread-local so it's inherently safe. This is well-designed for concurrent use.
- **Suggested Fix:** None needed -- this is already correct.

## Summary

The most significant free-threading concern is the lazy-init caching pattern in `DEFINE_CACHING_PYTHON_IMPORT_GETTER` (issue 4), which has a data race in the pre-pybind-2.13 code path when multiple threads simultaneously trigger the import cache. The several uses of `PyDict_GetItemWithError` with borrowed references (issue 6) are also notable under free-threading since concurrent dict mutations could invalidate the borrowed pointer. The global `PyObject*` variables (`THPVariableClass`, `ParameterClass`, `THPVariableFunctionsModule`, `device_to_py_class_`) follow a set-once-during-init-read-many pattern that is likely safe in practice due to Python import serialization, but lacks formal memory ordering guarantees.
