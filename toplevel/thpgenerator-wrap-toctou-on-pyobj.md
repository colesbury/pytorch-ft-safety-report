# `THPGenerator_Wrap` TOCTOU on `pyobj()` / `set_pyobj()`

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 1
- **Component:** Generator.cpp

- **Shared state:** The `pyobj_` field on `c10::GeneratorImpl` -- a plain
  (non-atomic) `PyObject*` pointer (c10/core/GeneratorImpl.h:89-94). This
  field is read by `pyobj()` and written by `set_pyobj()` without any
  synchronization.
- **Writer(s):**
  - `THPGenerator_NewWithVar()` calls `set_pyobj(g->cdata, obj)` (line 407)
    after allocating a new Python wrapper for a Generator.
  - `THPGenerator_dealloc()` calls `self->cdata.set_pyobj(nullptr)` (line 36)
    when the Python wrapper is deallocated.
- **Reader(s):**
  - `THPGenerator_Wrap()` calls `pyobj(gen)` (line 381) to check if a Python
    wrapper already exists. If it returns non-null, it increfs and returns
    the existing wrapper. If null, it creates a new one via
    `THPGenerator_NewWithVar`.
- **Race scenario:** Thread A and Thread B both hold a reference to the same
  `at::Generator` (e.g., `torch.default_generator` cloned or shared). Both
  call `THPGenerator_Wrap()` concurrently. Thread A reads `pyobj(gen)` and
  gets `nullptr`. Thread B also reads `pyobj(gen)` and gets `nullptr`. Both
  proceed to call `THPGenerator_NewWithVar`, which creates a new Python
  wrapper and calls `set_pyobj`. Now two different Python objects both claim
  to be the wrapper for the same Generator. The second `set_pyobj` overwrites
  the first, and the first Python object's dealloc will call
  `set_pyobj(nullptr)`, clearing the mapping even though the second object
  is still alive. Subsequent wraps will create yet another object, breaking
  object identity.
  Additionally, `pyobj_` is a non-atomic pointer, so reading it while another
  thread writes it is a data race (undefined behavior), though on most
  architectures pointer read/write is naturally atomic.
- **Suggested fix:** Use the same `PyObjectPreservation::init_once` pattern
  that `THPStorage_Wrap` already uses (Storage.cpp:84-91). That pattern uses
  `PyUnstable_TryIncRef` and `compare_exchange` to atomically establish the
  PyObject mapping, handling the race where two threads try to wrap
  simultaneously.
