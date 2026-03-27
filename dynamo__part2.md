# Free-Threading Safety Audit: `torch/csrc/dynamo/` (Part 2)

## Files Reviewed
- `dynamo/framelocals_mapping.h`
- `dynamo/guards.cpp`
- `dynamo/guards.h`
- `dynamo/init.cpp`
- `dynamo/init.h`
- `dynamo/python_compiled_autograd.cpp`
- `dynamo/python_compiled_autograd.h`
- `dynamo/stackref_bridge.h`
- `dynamo/utils.cpp`
- `dynamo/utils.h`

## Overall Risk: **Medium-High**

The most serious issues are concentrated in `python_compiled_autograd.cpp` (global mutable `PyObject*` pointers and a global `RuntimeState*`) and in `guards.cpp` (global mutable `dict_version_map`, `dict_to_guard_managers`, and `global_dict_version_id`). The compiled-autograd path has a static mutex that serializes execution, which mitigates some but not all of its issues. The dict-version tracking and dict-watcher infrastructure in guards.cpp has no synchronization at all.

---

## Issues

### Issue 1 — Global mutable `PyObject*` pointers in `python_compiled_autograd.cpp`

**Severity:** High
**Category:** Static/global mutable `PyObject*`

```cpp
// python_compiled_autograd.cpp:54-56 (anonymous namespace)
PyObject* the_autograd_compiler = nullptr;
int default_dyn_type_int = 0;
PyObject* python_verbose_logger = nullptr;
```

`set_autograd_compiler` (line 1252) and `set_verbose_logger` (line 652) write to these globals from Python-callable functions. `the_autograd_compiler` is also read by `_compiled_autograd_impl` (line 980) and `compiled_autograd` (line 1200). While `compiled_autograd` takes a static mutex, `set_autograd_compiler` does not acquire that same mutex before writing `the_autograd_compiler`. A concurrent call to `set_autograd_compiler` while another thread is inside `compiled_autograd` (which reads `the_autograd_compiler` after acquiring the mutex) is a data race on the pointer itself.

`python_verbose_logger` is set/read without any synchronization. `set_verbose_logger` stores a borrowed reference (no `Py_INCREF`), so the object can be freed by another thread, leaving a dangling pointer that `VerboseLogger::maybe_create()` (line 398) will dereference.

### Issue 2 — Global mutable `active_rstate` in `python_compiled_autograd.cpp`

**Severity:** Medium
**Category:** Static mutable C++ state in Python-facing functions

```cpp
// python_compiled_autograd.cpp:96
static RuntimeState* active_rstate;
```

Written by `RuntimeStateGuard` constructor/destructor (lines 98-110) and read by `call_cpp_tensor_pre_hooks` (line 123). Although the compiled-autograd path is serialized by a mutex, `call_cpp_tensor_pre_hooks` is exposed as a Python C-API method (line 674) and could be invoked from any thread at any time, racing on the raw pointer.

### Issue 3 — `CacheNode::root()` static singleton in `python_compiled_autograd.cpp`

**Severity:** Medium
**Category:** Static mutable C++ state / lazy init

```cpp
// python_compiled_autograd.cpp:464-466
static CacheNode* root() {
    static CacheNode _root;
    return &_root;
}
```

The static local initialization is thread-safe in C++11+, but the returned `CacheNode` is then mutated via `lookup()` (line 949), `check_dynamic_sizes()` (line 976), `clear()` (line 638), etc. `clear_cache` (line 636) is a Python-callable function that calls `CacheNode::root()->clear()` without acquiring the compiled-autograd mutex. This races with `_compiled_autograd_impl` which traverses and mutates the same tree.

### Issue 4 — `dict_version_map` and `global_dict_version_id` in `guards.cpp`

**Severity:** High
**Category:** Static/global mutable C++ state in Python-facing functions

```cpp
// guards.cpp:877-880 (IS_PYTHON_3_12_PLUS block)
static std::unordered_map<PyObject*, uint64_t> dict_version_map;
static int dict_version_watcher_id;
static int dict_recursive_tag_watcher_id;
static uint64_t global_dict_version_id = 1;
```

`dict_version_watch_callback` (line 881) is called from CPython's dict watcher infrastructure whenever a watched dict is mutated. It modifies `dict_version_map` and increments `global_dict_version_id`. `get_dict_version_unchecked` (line 896) also reads and writes `dict_version_map`. Under free-threading, dict mutations on different threads will invoke the watcher callback concurrently, producing data races on the `unordered_map` and the counter. This is unsynchronized and can corrupt the map.

### Issue 5 — `dict_to_guard_managers` global in `guards.cpp`

**Severity:** High
**Category:** Static/global mutable C++ state

```cpp
// guards.cpp:1640
std::unordered_map<PyObject*, std::list<GuardManager*>> dict_to_guard_managers;
```

This global map is mutated from:
- `GuardManager::watch_dict_pointers` (line 3427) — during guard setup
- `GuardManager::unwatch_all_saved_dict_pointers` (lines 3455-3468) — during guard manager deallocation
- `dict_recursive_tag_watch_callback` (line 4443) — from CPython dict watcher on any dict mutation

The dict watcher callback can fire from any thread that mutates a watched dict. There is no locking, so concurrent mutations to `dict_to_guard_managers` are a data race.

### Issue 6 — Lazy-init `static PyObject*` interned strings in `python_compiled_autograd.cpp`

**Severity:** Low
**Category:** Lazy init patterns

```cpp
// python_compiled_autograd.cpp:803
static PyObject* method_name = PyUnicode_InternFromString("begin_capture");
// python_compiled_autograd.cpp:848
static PyObject* log_compile_reasons = PyUnicode_InternFromString("log_compile_reasons");
// python_compiled_autograd.cpp:857
static PyObject* method_name = PyUnicode_InternFromString("end_capture");
// python_compiled_autograd.cpp:874
static PyObject* method_name = PyUnicode_InternFromString("close");
```

These are static local `PyObject*` variables initialized on first call. In C++, the initialization is thread-safe (magic statics), but `PyUnicode_InternFromString` calls into the Python runtime. Under free-threading, the Python runtime call itself should be safe with the per-object locks, but the pattern of storing a raw `PyObject*` in a static without reference management is fragile. In practice, interned strings are immortalized in CPython 3.12+, so this is low severity.

### Issue 7 — Lazy-init `static PyObject*` interned strings in `guards.cpp`

**Severity:** Low
**Category:** Lazy init patterns

```cpp
// guards.cpp:2104
static PyObject* current_device_str = PyUnicode_InternFromString("CURRENT_DEVICE");
// guards.cpp:2655
static PyObject* dynamic_indices_str = PyUnicode_InternFromString("_dynamo_dynamic_indices");
// guards.cpp:2665
static PyObject* issubset_str = PyUnicode_InternFromString("issubset");
```

Same pattern as Issue 6. These are interned strings cached in static locals. Low risk because interned strings are immortalized in modern CPython, and C++ guarantees thread-safe initialization of static locals.

### Issue 8 — Static `py::object` lazy init (pre-pybind 2.13 path) in `guards.cpp`

**Severity:** Medium
**Category:** Lazy init patterns

```cpp
// guards.cpp:4480-4485 (#else path of IS_PYBIND_2_13_PLUS)
static py::object guard_manager_enum_class =
    py::module_::import("torch._dynamo.guards").attr("GuardManagerType");
static py::object base_guard_manager_enum =
    guard_manager_enum_class.attr("GUARD_MANAGER");
static py::object dict_guard_manager_enum =
    guard_manager_enum_class.attr("DICT_GUARD_MANAGER");
```

These `static py::object` variables are lazily initialized on first call. While C++ guarantees thread-safe initialization of local statics, the `py::module_::import()` and `.attr()` calls involve Python runtime operations that may not be safe to execute concurrently without the GIL. The `#if IS_PYBIND_2_13_PLUS` path correctly uses `gil_safe_call_once_and_store`, but the fallback path does not.

### Issue 9 — `set_verbose_logger` stores borrowed reference in `python_compiled_autograd.cpp`

**Severity:** Medium
**Category:** Borrowed references from Python containers

```cpp
// python_compiled_autograd.cpp:652-665
static PyObject* set_verbose_logger(PyObject* dummy, PyObject* args) {
    ...
    if (logger == Py_None) {
        python_verbose_logger = nullptr;
    } else {
        python_verbose_logger = logger;  // borrowed reference, no Py_INCREF
    }
    ...
}
```

The `python_verbose_logger` pointer is stored without incrementing the reference count. Under free-threading, the Python object could be garbage collected from another thread, making the pointer dangling. `VerboseLogger::maybe_create()` and `PythonLogger::log()` then dereference it without any protection.

### Issue 10 — `static bool flag` lazy init for `validate_outputs` binding in `python_compiled_autograd.cpp`

**Severity:** Low
**Category:** Lazy init patterns

```cpp
// python_compiled_autograd.cpp:1077-1088
static bool flag [[maybe_unused]] = [&]() {
    ...
    bind_function(py_compiler.get(), "validate_outputs", ...);
    return true;
}();
```

This captures `py_compiler` by reference in a lambda used for one-time static initialization. The C++ runtime guarantees this runs exactly once, but the lambda captures a local variable by reference; if two threads race on first entry, the second thread will block until the first completes, which is correct. However, `bind_function` interacts with the Python runtime, and the `py_compiler` reference captured belongs to the calling thread. This is low risk because compiled autograd is serialized by a mutex.

## Summary

| # | Issue | Severity | File | Line(s) |
|---|-------|----------|------|---------|
| 1 | Global mutable `the_autograd_compiler`, `default_dyn_type_int`, `python_verbose_logger` | High | `python_compiled_autograd.cpp` | 54-56 |
| 2 | Global mutable `active_rstate` | Medium | `python_compiled_autograd.cpp` | 96 |
| 3 | `CacheNode::root()` unsynchronized mutations | Medium | `python_compiled_autograd.cpp` | 464, 638, 949, 976 |
| 4 | `dict_version_map` and `global_dict_version_id` unsynchronized | High | `guards.cpp` | 877-880, 881-912 |
| 5 | `dict_to_guard_managers` unsynchronized | High | `guards.cpp` | 1640, 3427, 3455-3468, 4443 |
| 6 | Lazy-init interned strings (compiled_autograd) | Low | `python_compiled_autograd.cpp` | 803, 848, 857, 874 |
| 7 | Lazy-init interned strings (guards) | Low | `guards.cpp` | 2104, 2655, 2665 |
| 8 | Static `py::object` lazy init (pre-pybind 2.13) | Medium | `guards.cpp` | 4480-4485 |
| 9 | `set_verbose_logger` stores borrowed reference | Medium | `python_compiled_autograd.cpp` | 652-665 |
| 10 | `static bool flag` lazy init for validate_outputs | Low | `python_compiled_autograd.cpp` | 1077-1088 |

## Notes

- `guards.h` declares `thread_local bool tls_is_in_mode_without_ignore_compile_internals` and the corresponding getter/setter, which is correctly thread-local and safe.
- `framelocals_mapping.h` defines a per-instance struct with no shared mutable state; safe.
- `stackref_bridge.h` is a simple function declaration; no state.
- `utils.cpp` and `utils.h` have no mutable global state.
- `init.cpp` performs module initialization which is typically single-threaded during import. The `_PyOpcode_Caches_vec` global is populated once during init and read-only afterward, which is acceptable.
- `init.h` is a simple header with no state.
- `python_compiled_autograd.h` is a simple header with no state.
- The `compiled_autograd` function (line 1200) acquires a `static std::mutex` before entering `_compiled_autograd_impl`, which mitigates many of the compiled-autograd issues. However, `set_autograd_compiler`, `clear_cache`, `is_cache_empty`, and `set_verbose_logger` do **not** acquire this mutex, creating windows for races.
