# Profiler py_gc_callback global pointer race

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 1
- **Component:** profiler_python.cpp

## Shared state

```cpp
static PyObject* py_gc_callback = nullptr;
```

A file-scope global `PyObject*` that stores the GC callback function object
registered with Python's `gc.callbacks`.

## Writers

- `PythonTracer::register_gc_callback()` sets `py_gc_callback` to a newly
  created `PyCFunction` (line 1146).
- `unregister_gc_callback()` reads and clears `py_gc_callback` (lines
  1111-1120).

## Readers

- `unregister_gc_callback()` reads `py_gc_callback` to find it in
  `gc.callbacks` and to `Py_XDECREF` it.
- Python's GC can trigger the callback (and thus access the `PyCapsule`
  self argument, which references the `PythonTracer*` instance) from any
  thread.

## Race scenario

Under free-threading, Python's garbage collector can run on any thread. If
thread A triggers a GC cycle, Python invokes `py_gc_callback`. Meanwhile,
thread B calls `stop()` which calls `unregister_gc_callback()`, which
`Py_XDECREF`s `py_gc_callback` and sets it to `nullptr`. If GC finishes
*after* the XDECREF, the callback's capsule may reference a freed
`PythonTracer`.

More fundamentally, the pattern of storing a single global `PyObject*` for
GC callbacks means that only one PythonTracer can register a GC callback.
If a second profiling session were created (on a different thread), it would
overwrite the global.

## Suggested fix

Make the GC callback registration per-tracer rather than global. Store the
callback reference in the `PythonTracer` instance. Use proper
synchronization when accessing it, or ensure registration/unregistration
are serialized (e.g., under the active_lock_).
