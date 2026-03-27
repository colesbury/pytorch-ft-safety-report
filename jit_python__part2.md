# Free-Threading Safety Audit: jit/python (Part 2)

## Overall Risk: LOW

Most files in this batch are JIT compiler infrastructure (sugared values, tree views, script compilation) that run with the GIL held via pybind11 dispatch. The global mutable state patterns are limited and mostly already handled correctly (atomic function pointers in tracer, thread-local in update_graph_executor_opt and utf8_decoding_ignore). The main issues are a legacy lazy-init pattern behind a preprocessor `#else` branch and global state set during module initialization without synchronization.

---

## Issues

### Issue 1: Legacy static py::object lazy initialization in extractTensors

- **Category:** Lazy init / static mutable PyObject*
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/jit/python/python_ivalue.h`
- **Lines:** 64-66
- **Description:** The `#else` branch (when `IS_PYBIND_2_13_PLUS` is not defined) uses a leaked `static py::object` for lazy initialization of `extractorFn`:
  ```cpp
  static py::object& extractorFn = *new py::object(
      py::module::import("torch._jit_internal").attr("_extract_tensors"));
  ```
  Static local initialization in C++ is guaranteed to be thread-safe for the initialization itself (C++11 magic statics), but the `py::module::import` call and attribute lookup involve Python C API operations. Under free-threading, two threads could race to initialize this simultaneously -- while the C++ runtime ensures only one executes the initializer, the Python calls within that initializer are not protected by any Python-level critical section. However, the `#if IS_PYBIND_2_13_PLUS` branch uses `gil_safe_call_once_and_store` which is the correct free-threading-safe pattern, and modern pybind11 versions (>= 2.13) should always take that path.
- **Fix:** Ensure `IS_PYBIND_2_13_PLUS` is always defined when building for free-threaded Python, or remove the legacy `#else` branch entirely if pybind11 >= 2.13 is the minimum supported version.

### Issue 2: Global function pointer registration during module init (python_tracer.cpp)

- **Category:** Module init setting global state
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/jit/python/python_tracer.cpp`
- **Lines:** 209-210
- **Description:** `initPythonTracerBindings` sets global function pointers via `setPythonCallstack(_pythonCallstack)` and `setRecordSourceLocation(pythonRecordSourceLocation)`. These global function pointers are stored in `std::atomic` variables (confirmed in `tracer.cpp`), so the stores themselves are atomic. However, the `_tracer_warn_use_python` binding (line 248) lazily sets the warn callback when called from Python: `tracer::setWarn(pythonWarn)`. This is also stored atomically. No actual race exists here.
- **Fix:** No fix needed. The underlying storage is already atomic.

### Issue 3: ConcretePyObjectHolder::getPyObject returns raw PyObject* without protection

- **Category:** Borrowed references from Python containers
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/jit/python/python_ivalue.h`
- **Lines:** 25-27
- **Description:** `getPyObject()` returns a raw `PyObject*` from the internal `py_obj_` without acquiring the GIL or incrementing the reference count. Under free-threading, if the `ConcretePyObjectHolder` is destroyed on another thread while the returned pointer is still in use, the PyObject could be freed. However, the caller holds an `intrusive_ptr` to the `PyObjectHolder`, so as long as the holder is alive the `PyObject*` remains valid. The risk is if the raw pointer escapes the lifetime of the holder.
- **Fix:** Callers should ensure they hold the `intrusive_ptr` for the duration of use of the raw `PyObject*`. This is a design-level concern rather than a bug in this specific code.

### Issue 4: Static local unordered_set in CUDAPythonModuleValue::attr

- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/jit/python/python_sugared_value.cpp`
- **Lines:** 220-231
- **Description:** `CUDAPythonModuleValue::attr` creates a `const` `static` `std::unordered_set<std::string>` inside the function. Because it is `const` after initialization, and C++11 guarantees thread-safe initialization of function-local statics, this is safe. No issue.
- **Fix:** None needed.

### Issue 5: thread_local state in update_graph_executor_opt.cpp and utf8_decoding_ignore.cpp

- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** High
- **File:** `torch/csrc/jit/python/update_graph_executor_opt.cpp`, `torch/csrc/jit/python/utf8_decoding_ignore.cpp`
- **Lines:** `update_graph_executor_opt.cpp:6`, `utf8_decoding_ignore.cpp:6`
- **Description:** Both files use `thread_local` for their state (`kOptimize`, `kIgnore`). This is inherently thread-safe since each thread has its own copy. However, under free-threading, the semantic intent may be to have a process-wide setting. If Python code sets `_set_graph_executor_optimize(True)` in one thread, other threads will not see the change. This is an existing semantic choice, not a race condition.
- **Fix:** None needed for thread safety. If process-wide semantics are desired, these would need to become `std::atomic<bool>`.

### Issue 6: registerNamedTuple races on global type registry

- **Category:** Lazy init / global mutable state
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/jit/python/python_sugared_value.cpp`
- **Lines:** 1010-1053
- **Description:** `registerNamedTuple` calls `get_python_cu()->get_type(qualifiedName)` and then conditionally `get_python_cu()->register_type(tt)`. Under free-threading, two threads could simultaneously try to register the same NamedTuple type, leading to a potential double-registration or TOCTOU race. However, this function is part of the JIT compiler which typically runs under Python's compilation locks, and `get_python_cu()` likely has its own internal synchronization. The risk in practice is very low.
- **Fix:** If this path can be reached concurrently, add synchronization around the check-then-register pattern, or make the compilation unit's `register_type` idempotent for duplicate registrations.

---

## Files Reviewed (No Issues Found)

- **`jit/python/python_ir.h`**: Pure declarations plus `ConcretePythonOp` struct. No global mutable state.
- **`jit/python/python_list.cpp`**: Pybind11 binding code for `ScriptList`. All operations go through pybind11's method dispatch. No global/static mutable state.
- **`jit/python/python_list.h`**: Class definitions with no static/global state.
- **`jit/python/python_sugared_value.h`**: Class declarations. No global mutable state.
- **`jit/python/python_tracer.h`**: Pure declarations.
- **`jit/python/python_tree_views.cpp`**: Pybind11 binding code for AST tree views. No global/static mutable state.
- **`jit/python/python_tree_views.h`**: Single function declaration.
- **`jit/python/script_init.cpp`**: Large file with pybind11 bindings. All state is either local to binding lambdas or accessed through existing synchronized APIs (`get_python_cu()`, etc.). The `magic_method_names` array is `constexpr` and immutable.
- **`jit/python/script_init.h`**: Single function declaration.
- **`jit/python/update_graph_executor_opt.h`**: Pure declarations.
- **`jit/python/utf8_decoding_ignore.h`**: Pure declarations.
