# Free-Threading Safety Audit: profiler_python

**Overall Risk: MEDIUM**

## Files Reviewed
- `profiler/python/combined_traceback.cpp`
- `profiler/python/combined_traceback.h`
- `profiler/python/init.cpp`
- `profiler/python/init.h`
- `profiler/python/pybind.h`

## Detailed Findings

### Issue 1 -- Static mutable `to_free_frames` and mutex (python/combined_traceback.cpp)

**Severity: LOW**
**Location:** `python/combined_traceback.cpp`, lines ~24-25

```cpp
static std::mutex to_free_frames_mutex;
static std::vector<CapturedTraceback::PyFrame> to_free_frames;
```

These are correctly protected by `to_free_frames_mutex` in all access paths (`gather`, `release`, `freeDeadCapturedTracebackFrames`). The mutex-based synchronization is valid for free-threading.

### Issue 2 -- GIL acquisition in PythonTraceback::gather (python/combined_traceback.cpp)

**Severity: MEDIUM**
**Location:** `python/combined_traceback.cpp`, lines ~44-63

```cpp
std::vector<CapturedTraceback::PyFrame> gather() override {
    std::vector<CapturedTraceback::PyFrame> frames;
    py::gil_scoped_acquire acquire;
    ...
    PyFrameObject* f = PyEval_GetFrame();
    Py_XINCREF(f);
    while (f) {
      frames.emplace_back(
          CapturedTraceback::PyFrame{PyFrame_GetCode(f), PyFrame_GetLasti(f)});
      auto f_back = PyFrame_GetBack(f);
      Py_XDECREF(f);
      f = f_back;
    }
    return frames;
}
```

Under free-threading, acquiring the GIL via `py::gil_scoped_acquire` becomes acquiring the mutex-based GIL equivalent. However, `PyEval_GetFrame()` returns a borrowed reference to the current thread's frame. Walking the frame stack is inherently thread-local in CPython (each thread has its own frame stack), so this is safe even under free-threading. The `PyFrame_GetCode` / `PyFrame_GetBack` calls return new references (in 3.11+), which are properly managed.

However, the `canGather()` check uses `PyGILState_Check()` and `PyGILState_GetThisThreadState()`, which under free-threading (3.13t+) may behave differently. `PyGILState_Check` always returns 1 in free-threading builds, so all threads will attempt to gather. This could cause `py::gil_scoped_acquire` to be called from threads that were never associated with Python, potentially leading to issues.

**Recommendation:** For free-threading, the `canGather()` logic should check `_PyThreadState_GET() != NULL` or use `PyGILState_GetThisThreadState()` which still works in 3.13t to check if the thread has a Python thread state.

### Issue 3 -- `gatherForwardTraceback` uses Python C API without critical sections (python/combined_traceback.cpp)

**Severity: MEDIUM**
**Location:** `python/combined_traceback.cpp`, lines ~134-212

This method acquires the GIL and then accesses Python dict objects:

```cpp
PyObject* dict = metadata->dict();
if (!dict || !PyDict_Check(dict)) { ... }
PyObject* traceback = nullptr;
if (PyDict_GetItemStringRef(dict, ..., &traceback) < 0) { ... }
Py_ssize_t size = PyList_Size(traceback);
for (Py_ssize_t i = 0; i < size; ++i) {
    PyObject* item = PyList_GetItem(traceback, i); // borrowed reference
```

Under free-threading, `PyList_GetItem` returns a borrowed reference. Another thread could concurrently modify the list, invalidating the borrowed reference. `PyDict_GetItemStringRef` is safe (returns a new reference), but `PyList_GetItem` should be replaced with `PyList_GetItemRef` (3.13+) or protected with `Py_BEGIN_CRITICAL_SECTION`.

**Recommendation:** Use `Py_BEGIN_CRITICAL_SECTION(traceback)` around the list iteration, or switch to `PyList_GetItemRef`.

### Issue 4 -- `appendSymbolized` accesses PyCodeObject fields directly (python/combined_traceback.cpp)

**Severity: LOW**
**Location:** `python/combined_traceback.cpp`, lines ~103-128

```cpp
auto f_code = (PyCodeObject*)f.code;
py::handle filename = f_code->co_filename;
py::handle funcname = f_code->co_name;
auto lineno = PyCode_Addr2Line(f_code, f.lasti);
```

Code objects are immutable in CPython, so `co_filename` and `co_name` do not change after creation. This is safe under free-threading.

### Issue 5 -- `RecordFunctionFast` Python type methods (python/init.cpp)

**Severity: LOW**
**Location:** `python/init.cpp`, lines ~217-322

The `RecordFunctionFast_enter` and `RecordFunctionFast_exit` methods access `self->guard`, `self->name`, `self->input_values`, `self->keyword_values`. These are per-instance members. Python's `__enter__`/`__exit__` protocol is typically called from a single thread per context manager instance. Under free-threading, if the same `RecordFunctionFast` object were used from multiple threads simultaneously, there would be races on `self->guard`, but this would be a misuse of the context manager pattern. The risk is low.

The `RecordFunctionFast_enter` method also calls `PySequence_Fast`, `PyDict_Next`, `PyList_GetItem`, and `PyList_Size` which involve borrowed references. Under free-threading these could race if other threads modify the same containers. However, the input containers are typically not shared across threads.

### Issue 6 -- Static `PyTypeObject` definitions (python/init.cpp)

**Severity: LOW**
**Location:** `python/init.cpp`, lines ~41-81, 731-742

`THPCapturedTracebackType` and `RecordFunctionFast_Type` are static `PyTypeObject` structs initialized once during module init via `PyType_Ready`. After initialization they are immutable. Safe.

### Issue 7 -- `PyErr_Fetch` / `PyErr_Restore` pattern (python/combined_traceback.cpp)

**Severity: LOW**
**Location:** `python/combined_traceback.cpp`, lines ~169-176

```cpp
PyErr_Fetch(&exc_type, &exc_value, &exc_tb);
...
PyErr_Restore(exc_type, exc_value, exc_tb);
```

The per-thread error indicator is inherently thread-local in CPython (each thread state has its own `curexc_*`). These calls are safe under free-threading since they operate on the current thread's exception state.

## Summary

The main concerns are: (1) `PyList_GetItem` usage returning borrowed references that could be invalidated under concurrent modification (Issue 3), and (2) the `canGather()` heuristic for detecting Python-capable threads may need adjustment for free-threading builds (Issue 2). The deferred frame freeing mechanism (`to_free_frames`) is correctly mutex-protected. Most other patterns in these files are safe.
