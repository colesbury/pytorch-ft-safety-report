# `Py_MOD_GIL_NOT_USED` is the root cause of S1-S4

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 1
- **Shared state:** All module-level functions.
- **Race scenario:** By declaring `Py_MOD_GIL_NOT_USED`, all five exported C
  methods can be called without the GIL on 3.14t. None of them have adequate
  internal synchronization for concurrent use. Removing this declaration would
  cause CPython to acquire the GIL before entering any module method, which
  combined with the `pybind11::gil_scoped_acquire` in `compiled_autograd()`
  would serialize all relevant accesses and eliminate S1-S4 without any other
  code changes.
- **Suggested fix:** Remove `Py_MOD_GIL_NOT_USED` until the module methods
  have proper internal synchronization. This is the highest-leverage single
  fix.
