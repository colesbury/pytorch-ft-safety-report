# `py_gc_callback` global pointer race between registration and GC

- **Status:** Open
- **Severity:** Significant
- **Component:** autograd/profiler_python.cpp

- **Shared state:** `py_gc_callback` -- a file-scope `static PyObject*`
  (line 995) used to track the GC callback for later unregistration.
- **Writer(s):**
  - `register_gc_callback()` (line 1146): sets `py_gc_callback` to the newly
    created `PyCFunction`.
  - `unregister_gc_callback()` (line 1119-1120): decrefs and sets to `nullptr`.
- **Reader(s):**
  - `unregister_gc_callback()` (line 1111): reads `py_gc_callback` to find and
    remove it from `gc.callbacks`.
- **Race scenario:** If two `PythonTracer` instances are created and destroyed
  (even sequentially but overlapping with GC), the global `py_gc_callback`
  is shared state without synchronization. More concretely: under
  free-threading, `register_gc_callback` and `unregister_gc_callback` could
  overlap if one thread is destroying a tracer while another is creating one.
  The `active_lock_` atomic prevents two active tracers, but the GC callback
  registration/unregistration is not protected by that lock (it happens after
  the lock check in the constructor and in `stop()`).

  The global variable pattern also means only one tracer's callback can be
  tracked. If `register_gc_callback` overwrites `py_gc_callback` while the
  previous value is still in `gc.callbacks`, the old callback leaks and the
  old tracer's capsule pointer becomes a dangling reference.
- **Suggested fix:** Make `py_gc_callback` an instance member of
  `PythonTracer` instead of a file-scope global. The `PyCapsule` already
  carries the `PythonTracer*`, so unregistration can use the instance
  member directly.
