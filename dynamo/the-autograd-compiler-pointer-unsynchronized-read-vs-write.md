# `the_autograd_compiler` pointer: unsynchronized read vs. write

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 1 (any thread can call `set_autograd_compiler`)
- **Shared state:** `the_autograd_compiler` (file-static `PyObject*`, line 54)
- **Writer(s):** `set_autograd_compiler()` (line 1252) writes the pointer:
  either sets it to `nullptr` (disable) or to a new `PyObject*` with
  `Py_INCREF` (enable). No lock is held.
- **Reader(s):** `_compiled_autograd_impl()` (line 980) reads the pointer to
  call `PyObject_CallNoArgs(the_autograd_compiler)`. This runs under the
  compiled autograd mutex and GIL, but the writer does not hold either.
- **Race scenario:** Thread A is in `_compiled_autograd_impl` about to call
  `PyObject_CallNoArgs(the_autograd_compiler)`. Thread B calls
  `set_autograd_compiler(None, 0)` from Python. On 3.14t, Thread B does not
  need the GIL to enter the function. Thread B sets
  `the_autograd_compiler = nullptr` and returns the prior `PyObject*` to the
  caller, which decrefs it. If Thread A has already loaded the pointer value
  but not yet completed the call, it calls into a freed object. Even on
  architectures where pointer writes are atomic, the refcount management is
  unsynchronized: the object can be freed before Thread A's call completes.
- **Suggested fix:** Have `set_autograd_compiler` acquire the compiled
  autograd mutex before modifying the pointer.
