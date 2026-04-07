# `python_verbose_logger` borrowed reference: use-after-free

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 1 (any thread can call `set_verbose_logger`)
- **Shared state:** `python_verbose_logger` (file-static `PyObject*`, line 56)
  -- stored as a **borrowed reference** (no `Py_INCREF`).
- **Writer(s):** `set_verbose_logger()` (line 652) stores the raw `PyObject*`
  directly, or sets it to `nullptr`. No reference count increment is performed.
- **Reader(s):** `VerboseLogger::maybe_create()` (line 398) reads the pointer.
  `PythonLogger::log()` (line 370) dereferences it. Both run inside the
  compiled backward path under the mutex.
- **Race scenario 1 (free-threading):** Thread A is inside a compiled backward,
  has captured `python_verbose_logger` into a `VerboseLogger` instance (line
  402). Thread B calls `set_verbose_logger(None)` which sets
  `python_verbose_logger = nullptr`. The Python logger object may be
  garbage-collected if the caller held the only reference (since no `Py_INCREF`
  was done). Thread A later calls `vlogger->log()` which dereferences the
  freed object.
- **Race scenario 2 (single-threaded):** Even without concurrency, the
  borrowed reference is fragile: if the Python caller of `set_verbose_logger`
  drops its reference, the stored pointer dangles. This is not a threading
  issue per se, but free-threading makes it much more likely to manifest.
- **Suggested fix:** `Py_XINCREF` the logger in `set_verbose_logger` and
  `Py_XDECREF` the old one. Use `THPObjectPtr` for the global. Protect reads
  with the mutex.
