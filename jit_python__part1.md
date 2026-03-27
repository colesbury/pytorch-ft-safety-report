# Free-Threading Safety Audit: jit/python (Part 1)

**Overall Risk: MEDIUM**

Most of the code in this batch consists of pybind11 module initialization functions that run once during import. The main concerns are: (1) a mutable static `global_print_source_ranges` that is read/written without synchronization, (2) the legacy `#else` path in `module_python.h` that unsafely caches `py::handle` in statics, (3) a static `py::handle` for the JIT exception class leaked during init, and (4) borrowed references from `PyDict_Items` in `python_arg_flatten.cpp`. The `pybind_utils.cpp` and `pybind_utils.h` files are largely stateless conversion utilities with only a `thread_local` flag, which is fine.

---

## Issues

### Issue 1 — Unsynchronized static `global_print_source_ranges`

| Field       | Value |
|-------------|-------|
| Category    | Static mutable C++ state in Python-facing functions |
| Severity    | Medium |
| Confidence  | High |
| File        | `torch/csrc/jit/python/python_ir.cpp` |
| Lines       | 25, 217, 223 |

**Description:**
`global_print_source_ranges` is a `static bool` that is both read (line 217, via `__repr__` on `Graph`) and written (line 223, via `set_global_print_source_ranges`) from Python-facing pybind methods. Without the GIL to serialize access, concurrent calls to `Graph.__repr__()` and `Graph.set_global_print_source_ranges()` from different threads constitute a data race.

**Fix:**
Change to `static std::atomic<bool> global_print_source_ranges{true}` and use `load()`/`store()` at the use sites.

---

### Issue 2 — Legacy static `py::handle` cache in `module_python.h` (non-pybind 2.13 path)

| Field       | Value |
|-------------|-------|
| Category    | Static/global mutable PyObject* |
| Severity    | Medium |
| Confidence  | Medium |
| File        | `torch/csrc/jit/python/module_python.h` |
| Lines       | 23-24, 49-52 |

**Description:**
When `IS_PYBIND_2_13_PLUS` is not defined, `as_module` and `as_object` use bare `static py::handle` variables initialized on first call via `py::module::import(...)`. Under free-threading, two threads could race to initialize these statics. While C++11 guarantees static local initialization is thread-safe for the C++ variable itself, the initialization expression calls Python (`py::module::import`) which performs arbitrary Python execution. This path could double-import or see a partially-constructed object.

The `IS_PYBIND_2_13_PLUS` path uses `py::gil_safe_call_once_and_store`, which is the correct pattern.

**Fix:**
Remove the `#else` branch entirely if pybind 2.13+ is always used, or replicate the `gil_safe_call_once_and_store` pattern under the `#else` path using `std::call_once` plus appropriate GIL management.

---

### Issue 3 — Static `py::handle exc` for JITException leaked during init

| Field       | Value |
|-------------|-------|
| Category    | Static/global mutable PyObject* |
| Severity    | Low |
| Confidence  | Medium |
| File        | `torch/csrc/jit/python/init.cpp` |
| Lines       | 180-181 |

**Description:**
```cpp
static py::handle exc =
    py::exception<JITException>(m, "JITException").release();
```
This static `py::handle` is initialized once during module init (`initJITBindings`), which is called from Python's single-threaded import mechanism. The handle itself is never mutated afterward, only read (line 199, `PyErr_SetString(exc.ptr(), ...)`). This is safe because module init is serialized, and `py::handle` is a simple wrapper around `PyObject*` with no ref-count management. However, if `initJITBindings` were ever called from multiple threads (unlikely but not structurally prevented), the static initialization could race.

**Fix:**
No immediate action needed if module init is guaranteed single-threaded. For defense-in-depth, annotate with a comment or use `std::call_once`.

---

### Issue 4 — `PyDict_Items` returns a new list iterated without critical section

| Field       | Value |
|-------------|-------|
| Category    | Python C API on shared objects without Py_BEGIN_CRITICAL_SECTION |
| Severity    | Low |
| Confidence  | Medium |
| File        | `torch/csrc/jit/python/python_arg_flatten.cpp` |
| Lines       | 57-64 |

**Description:**
In `flatten_rec`, `PyDict_Items(obj)` is called on a dict object. Under free-threading, if another thread mutates the dict concurrently (e.g., adding/removing keys), `PyDict_Items` could observe the dict in an inconsistent state. The returned list itself is a new object (safe to iterate), but the call to `PyDict_Items` reads the dict without holding a critical section on it.

This is a concern only if the same dict is shared across threads and mutated concurrently, which is a somewhat unusual pattern for JIT flatten inputs.

**Fix:**
Wrap the `PyDict_Items` call with `Py_BEGIN_CRITICAL_SECTION(obj)` / `Py_END_CRITICAL_SECTION()` (or the equivalent macro that is a no-op on GIL builds).

---

### Issue 5 — `ConcretePythonOp::cloneFrom` performs bare `Py_INCREF` without GIL awareness

| Field       | Value |
|-------------|-------|
| Category    | Python C API on shared objects without Py_BEGIN_CRITICAL_SECTION |
| Severity    | Low |
| Confidence  | Medium |
| File        | `torch/csrc/jit/python/python_ir.cpp` |
| Lines       | 114-125 |

**Description:**
`ConcretePythonOp::cloneFrom` calls `Py_INCREF(other->pyobj.get())` and `Py_INCREF(sa.get())` without acquiring the GIL or using any critical section. In the free-threading build, `Py_INCREF` on an object that could be concurrently decref'd by another thread is a data race on the refcount (in CPython 3.14t, `Py_INCREF` is made atomic, but calling it without GIL held is still questionable for callers that also touch the object state). More importantly, the function reads `other->cconv`, `other->pyobj`, and `other->scalar_args` without any synchronization, so if `other` is being mutated concurrently, those reads race.

**Fix:**
Acquire the GIL (`pybind11::gil_scoped_acquire`) at the top of `cloneFrom`, consistent with how other methods in this class (e.g., `name()`, `autogradFunction()`) already do.

---

### Issue 6 — `OpaqueObject` getPayload/setPayload unsynchronized

| Field       | Value |
|-------------|-------|
| Category    | Static mutable C++ state in Python-facing functions |
| Severity    | Low |
| Confidence  | Low |
| File        | `torch/csrc/jit/python/opaque_obj.h` |
| Lines       | 15-25 |

**Description:**
`OpaqueObject` stores a `py::object payload_` that can be read via `getPayload()` and written via `setPayload()`. The `__eq__` lambda (line 32-48) accesses `getPayload()` from potentially different threads. While the `__eq__` lambda does acquire the GIL for the comparison itself, it calls `getPayload()` before the GIL acquire (lines 34-35), meaning the `py::object` copy constructor runs without synchronization on the `payload_` field.

**Fix:**
Move the `py::gil_scoped_acquire` before the `getPayload()` calls, or add a mutex to `OpaqueObject` to protect concurrent access to `payload_`.

---

### Issue 7 — Static `register_opaque_obj_class` initialization in header

| Field       | Value |
|-------------|-------|
| Category    | Lazy init patterns |
| Severity    | Low |
| Confidence  | Low |
| File        | `torch/csrc/jit/python/opaque_obj.h` |
| Lines       | 28-77 |

**Description:**
`static auto register_opaque_obj_class = torch::class_<OpaqueObject>(...)` is defined at namespace scope in a header file. Each translation unit that includes this header gets its own copy (due to `static` linkage). The `torch::class_` registration modifies global registration tables. If multiple translation units initialize simultaneously during static init, the registration table modifications could race. In practice, static init order across TUs is serial within a single process, but dynamic library loading on some platforms could theoretically cause overlap.

**Fix:**
This is a pre-existing architectural pattern in PyTorch. Low priority. Could be moved to a .cpp file to ensure single registration.

---

### Issue 8 — `PyBytes_AsString` usage after releasing GIL

| Field       | Value |
|-------------|-------|
| Category    | Borrowed references from Python containers |
| Severity    | Low |
| Confidence  | Low |
| File        | `torch/csrc/jit/python/init.cpp` |
| Lines       | 1479-1481 |

**Description:**
```cpp
const char* data_str = PyBytes_AsString(data.ptr());
py::gil_scoped_release release;
return self.writeRecord(name, data_str, size);
```
`PyBytes_AsString` returns a pointer into the internal buffer of a Python bytes object. After releasing the GIL, if the `data` py::bytes object's refcount drops to zero on another thread, the buffer could be freed. However, because `data` is a pybind11 `py::bytes` object held in the lambda's scope, it keeps the reference alive. This is safe in practice but fragile under free-threading if the bytes object is somehow shared and mutated.

The code comment references CPython internals and notes this is okay. Under free-threading, the reasoning still holds as long as `data` remains alive, which it does.

**Fix:**
No fix strictly needed, but adding a comment about free-threading safety would help maintainability.

---

### Issue 9 — `setPrintHandler` sets global state from module init

| Field       | Value |
|-------------|-------|
| Category    | Module init setting global state concurrently |
| Severity    | Low |
| Confidence  | Low |
| File        | `torch/csrc/jit/python/init.cpp` |
| Lines       | 2391-2399 |

**Description:**
At the end of `initJITBindings`, `setPrintHandler` is called to install a Python-aware print handler, and an `atexit` callback resets it. The print handler itself acquires the GIL when called. The concern is that `setPrintHandler` mutates global state: if another thread is calling `prim::Print()` concurrently while the handler is being swapped, there is a brief window of inconsistency.

**Fix:**
This is during module init which is serialized. Low priority. If `setPrintHandler` is not already atomic/synchronized internally, it could be made so.

---

## Non-Issues (Confirmed Safe)

1. **`thread_local allow_numbers_as_tensors`** (`pybind_utils.cpp:23`): Correctly uses `thread_local` with RAII guard `ToIValueAllowNumbersAsTensors`. Thread-safe.

2. **`thread_local` in `jit_exception.cpp` and `utf8_decoding_ignore.cpp`**: `caughtOriginalMsg`, `caughtPythonClassName`, and `kIgnore` are all `thread_local`. Thread-safe.

3. **`thread_local inline_everything`** (`module.cpp:144`): Thread-safe.

4. **`std::atomic` for profiling/executor mode** (`getProfilingMode`, `getExecutorMode`, `getNumProfiledRuns`): Already use `std::atomic`. Thread-safe.

5. **`module_python.h` IS_PYBIND_2_13_PLUS path**: Uses `py::gil_safe_call_once_and_store`, which is the correct free-threading-safe lazy init pattern.

6. **`python_interpreter.cpp`**: `createPythonOperation` and its returned lambda both acquire GIL via `pybind11::gil_scoped_acquire`. The `RegisterOperators reg(...)` at namespace scope is static initialization, executed once. Safe.

7. **`python_dict.h` / `python_dict.cpp`**: `ScriptDict` wraps `c10::impl::GenericDict` and does not use global/static mutable state. The iterators hold their own copies of begin/end. No free-threading issues.

8. **`python_custom_class.cpp`**: Stateless pybind registration. No global mutable state. Safe.

9. **`pybind.h`**: Type caster definitions and `tuple_tail` utility. Purely functional. Safe.

10. **`python_arg_flatten.h`**: Data structure definitions only. Safe.
