# Free-Threading Audit: autograd__part4

**Files:**
- `torch/csrc/autograd/python_cpp_function.h`
- `torch/csrc/autograd/python_engine.cpp`
- `torch/csrc/autograd/python_engine.h`
- `torch/csrc/autograd/python_enum_tag.h`
- `torch/csrc/autograd/python_fft_functions.h`
- `torch/csrc/autograd/python_function.cpp`
- `torch/csrc/autograd/python_function.h`
- `torch/csrc/autograd/python_hook.cpp`
- `torch/csrc/autograd/python_hook.h`
- `torch/csrc/autograd/python_legacy_variable.cpp`
- `torch/csrc/autograd/python_legacy_variable.h`
- `torch/csrc/autograd/python_linalg_functions.h`
- `torch/csrc/autograd/python_nested_functions.h`
- `torch/csrc/autograd/python_nested_functions_manual.cpp`
- `torch/csrc/autograd/python_nn_functions.h`

**Overall Risk:** High

## Issues

### 1. Global mutable `THPFunctionClass` and `THPGradientEdgeClass` without synchronization
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_function.cpp`
- **Line(s):** 51-52
- **Description:** `THPFunctionClass` and `THPGradientEdgeClass` are global mutable `PyObject*` pointers. These are set during module initialization (from Python side) and read throughout the codebase. Under free-threading, if module init races with usage from another thread, this is a data race. In practice, module init typically completes before other threads use these, so the real risk is low, but this is technically a race.
- **Suggested Fix:** These are likely set once during module initialization. If so, ensure they are only read after initialization completes (e.g., via `std::atomic` or by relying on import lock serialization in CPython 3.14t).

### 2. Lazy-init pattern in `get_base_setup_context` with static `THPFunction_setup_context`
- **Category:** Lazy init pattern (`static PyObject* cache = nullptr; if (!cache) cache = ...`)
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/autograd/python_function.cpp`
- **Line(s):** 1307-1330
- **Description:** `THPFunction_setup_context` is a file-static `PyObject*` that is lazily initialized in `get_base_setup_context()`. Under free-threading, multiple threads can race on the null check and the assignment, leading to duplicate initialization, leaked references, or use of a partially-written pointer. This function is called from `THPFunction_apply` which is a hot path for custom autograd functions.
- **Suggested Fix:** Use `std::call_once` or an atomic pointer with acquire/release semantics to ensure the initialization runs exactly once. Alternatively, initialize this eagerly during module init.

### 3. `PyDict_GetItemString` returns borrowed reference in `_trace_post_record`
- **Category:** Borrowed references from Python containers
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_function.cpp`
- **Line(s):** 1077-1082
- **Description:** `PyDict_GetItemString` on `tp_dict` returns a borrowed reference. Under free-threading, another thread could modify `tp_dict` concurrently, invalidating the borrowed reference before `PyUnicode_AsUTF8` uses it. This is in the JIT tracing path which is not heavily used but is still a real race.
- **Suggested Fix:** Use `PyDict_GetItemStringRef` (available in 3.13+) which returns a strong reference, or use `PyObject_GetAttrString` instead which returns a new reference.

### 4. Static `_reinitialize_engine` flag without atomic access
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_engine.cpp`
- **Line(s):** 33, 47-53, 498
- **Description:** `_reinitialize_engine` is a plain `bool` that is written in `child_atfork()` (line 498) and read/written in `get_python_engine()` (lines 47-53). The existing comment acknowledges this is "best effort" and was only safe because of GIL serialization. Under free-threading, this is a data race on the flag itself. However, the fork handler context means threads should not exist after fork in the child, so the real risk is primarily around the read-check-write pattern in `get_python_engine()` being called from multiple threads concurrently (before/during fork).
- **Suggested Fix:** Make `_reinitialize_engine` a `std::atomic<bool>`. The existing comment already notes this is best-effort, but atomic access would at least eliminate the undefined behavior.

### 5. `static PythonArgParser` in `THPVariable_nested_tensor`
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/python_nested_functions_manual.cpp`
- **Line(s):** 14-16
- **Description:** `PythonArgParser` is declared `static` inside the function body. `PythonArgParser::parse()` likely mutates internal state (parsed argument caches). Under free-threading, concurrent calls to `torch.nested.nested_tensor()` from multiple threads would race on this shared mutable state. This is a common pattern across PyTorch C++ bindings.
- **Suggested Fix:** Either make `PythonArgParser` thread-safe internally, or use thread-local storage for mutable parser state.

### 6. `PyNode::compiled_args` uses `static PyObject* method_name` lazy init
- **Category:** Lazy init pattern
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/autograd/python_function.cpp`
- **Line(s):** 336-337
- **Description:** `static PyObject* method_name = PyUnicode_InternFromString("_compiled_autograd_key")` is a function-local static. In C++11+, function-local static initialization is thread-safe for the initialization itself (guaranteed by the standard). However, the resulting `PyObject*` is an interned string that may have Python-internal reference counting issues under free-threading if `PyUnicode_InternFromString` is not itself free-threading safe. In practice, CPython 3.14t has made interned strings immortal, so this is likely safe.
- **Suggested Fix:** No action strictly necessary if targeting CPython 3.14t where interned strings are immortal. If broader compatibility is needed, consider initializing during module init.

### 7. `_call_hooks` uses `PyTuple_SetItem` on `args` tuple potentially shared
- **Category:** Python C API on shared objects without critical section
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/autograd/python_hook.cpp`
- **Line(s):** 89
- **Description:** `PyTuple_SetItem(args, 0, res.release())` modifies the args tuple in-place. The tuple `args` is created locally in each caller (`operator()` methods) and is not shared with other threads, so this is likely safe. Noting it for completeness since `PyTuple_SetItem` on a shared tuple would be a race.
- **Suggested Fix:** No fix needed -- the tuple is locally owned.

## Summary

The most significant issue is the lazy-init pattern for `THPFunction_setup_context` (issue 2) which is on a hot path and has a classic TOCTOU race under free-threading. The `PyDict_GetItemString` borrowed reference (issue 3) and the `_reinitialize_engine` non-atomic flag (issue 4) are also real concerns. The `static PythonArgParser` pattern (issue 5) is a systemic issue across PyTorch bindings. The hook code in `python_hook.cpp` is notably well-prepared for free-threading, already using `Py_BEGIN_CRITICAL_SECTION`/`Py_END_CRITICAL_SECTION` around `PyDict_Next` iterations.
