# `guard_error_hook` and `guard_complete_hook` global PyObject* pointers

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `guard_error_hook` and `guard_complete_hook` -- global
  `PyObject*` pointers (declared in `eval_frame.c`).
- **Writer(s):**
  - `set_guard_error_hook()` -- uses `Py_XSETREF` to swap the pointer.
  - `set_guard_complete_hook()` -- manually swaps the pointer.
- **Reader(s):**
  - `lookup()` in `extra_state.cpp` reads `guard_error_hook` (line 173).
  - `dynamo__custom_eval_frame` reads `guard_complete_hook` (line 564).
- **Race scenario:** Thread A is in `lookup()` and reads `guard_error_hook`
  as non-null, then enters the error handling path using the pointer.
  Thread B calls `set_guard_error_hook(None)`, which `Py_XSETREF`s the
  pointer to NULL and decrefs the old object. Thread A now uses a freed
  `PyObject*`. This is a use-after-free.
  Same pattern for `guard_complete_hook`.
- **Tier:** **Tier 2** (hooks are typically set once during setup, but
  nothing prevents them from being changed during execution).
- **Suggested fix:** Use `std::atomic<PyObject*>` and ensure the old object's
  refcount is managed with appropriate ordering. Or protect with a lock.
