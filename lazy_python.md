# Free-Threading Safety Audit: lazy_python

## Overall Risk: LOW

## Files Audited
- `lazy/python/init.cpp`
- `lazy/python/init.h`
- `lazy/python/python_util.cpp`
- `lazy/python/python_util.h`

## Findings

### Issue 1: Global function pointer assignment without synchronization (init.cpp:336)

**Severity:** LOW
**Description:** `GetPythonFramesFunction() = GetPythonFrames;` assigns to a static `std::function` object during module initialization (`initLazyBindings`). The static is defined in `debug_util.cpp`. Module init is called once during interpreter startup, so concurrent writes are not expected, but if called from a secondary interpreter or if there is a race between initialization and first use, the write to the `std::function` is not atomic.
**Existing Protection:** Module init functions are called by CPython's import machinery which serializes module initialization. The default value (`NoPythonFrames`) is already safe.
**Recommendation:** Consider using `std::atomic` or `std::call_once` to ensure safe publication, though the current risk is minimal given single-interpreter, single-init semantics.

### Issue 2: PyEval_GetFrame without critical section (python_util.cpp:18, 34)

**Severity:** LOW
**Description:** `GetPythonFrameTop()` and `GetPythonFrames()` call `PyEval_GetFrame()` after acquiring the GIL via `pybind11::gil_scoped_acquire`. Under free-threading, `PyEval_GetFrame()` returns a borrowed reference to the current thread's frame. Since frame objects are per-thread, accessing another thread's frame is not an issue here -- each caller gets its own frame. The GIL acquire becomes a no-op under free-threading but the frame traversal is inherently thread-local.
**Existing Protection:** `pybind11::gil_scoped_acquire` is present. Frame objects are thread-local.
**Recommendation:** No changes needed. The frame walking is thread-local and safe.

### Issue 3: pybind11 bindings with GIL release (init.cpp:111, 123, 183)

**Severity:** LOW
**Description:** Several pybind11 lambda bindings use `pybind11::gil_scoped_release no_gil;` before calling into C++ backend operations (`SyncLiveTensorsGraph`, `WaitDeviceOps`, `SyncTensors`). These C++ operations use their own mutex-based synchronization and do not access Python objects, so releasing the GIL is safe and correct for free-threading.
**Existing Protection:** The C++ lazy tensor infrastructure uses internal mutexes for thread safety.
**Recommendation:** No changes needed.

### Issue 4: Static local in force_eager_fallback (ts_eager_fallback.cpp, called from init.cpp)

**Severity:** LOW
**Description:** `getLTCForceFallback()` returns a reference to a static `std::string`. Calls to `_set_force_fallback` and `_get_force_fallback` from Python could race, but these are configuration APIs typically called during setup, not during concurrent execution.
**Existing Protection:** None explicit (see lazy_core__part1 report for details).
**Recommendation:** If concurrent access is expected, protect with a mutex.

## Summary

The `lazy/python` files are a thin Python binding layer. Most heavy lifting is delegated to the C++ lazy tensor core, which has its own mutex-based synchronization. The Python frame-walking utilities are inherently thread-safe because they operate on per-thread state. The module initialization writes to a global function pointer but this occurs during import, which is serialized by CPython. No high-severity free-threading issues found.
