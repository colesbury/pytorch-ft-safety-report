# GC callback use-after-free of `PythonTracer` instance

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** autograd/profiler_python.cpp

- **Shared state:** The `PythonTracer*` pointer captured inside a `PyCapsule`
  that is bound to the `gc_event_callback` function (lines 1003-1011, 1145-1146).
  Also the global `py_gc_callback` pointer (line 995).
- **Writer(s):**
  - `register_gc_callback()` (line 1124): creates the PyCapsule containing
    `this` (the `PythonTracer*`) and appends it to `gc.callbacks`.
  - `unregister_gc_callback()` (line 1095): removes the callback from
    `gc.callbacks` and decrefs/nulls `py_gc_callback`.
  - `PythonTracer::~PythonTracer()`: destroys the `PythonTracer` and its
    `queue_` member.
- **Reader(s):**
  - `gc_event_callback()` (line 997): fired by Python's GC on whichever thread
    triggers a collection. Extracts `PythonTracer*` from the PyCapsule and
    calls `instance->queue_->getSubqueue()->emplace_gc_call(...)`.
- **Race scenario:** Thread A (main) calls `PythonTracer::stop()` then destroys
  the `PythonTracer`. `stop()` calls `unregister_gc_callback()` which removes
  the callback from `gc.callbacks`. However, under free-threading, Thread B
  could have already entered `gc_event_callback` (the GC triggered before the
  unregistration completed, or the list removal and the callback invocation
  race). Thread B then dereferences the `PythonTracer*` from the PyCapsule,
  which points to a destroyed object. This is a use-after-free on the
  `PythonTracer` and its `queue_` member.
- **Suggested fix:** Use a ref-counted weak pointer or an atomic flag inside the
  PyCapsule to signal invalidation. Alternatively, ensure the PyCapsule destructor
  is the mechanism that prevents stale access (e.g., set the capsule's pointer
  to nullptr under a lock, and check in the callback).
