# `active_rstate` global pointer: unsynchronized read from `call_cpp_tensor_pre_hooks`

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 2 (only relevant if `call_cpp_tensor_pre_hooks` is called
  from a thread other than the one running the compiled backward)
- **Shared state:** `active_rstate` (file-static `RuntimeState*`, line 96)
- **Writer(s):** `RuntimeStateGuard` constructor (line 99) sets
  `active_rstate = _state.get()`; destructor (line 107) sets
  `active_rstate = nullptr`. These run inside `compiled_autograd()` under the
  mutex.
- **Reader(s):** `call_cpp_tensor_pre_hooks()` (line 113) reads
  `active_rstate` and dereferences it. This is a `METH_VARARGS` method on a
  `Py_MOD_GIL_NOT_USED` module, callable without the GIL from any thread.
- **Race scenario:** Thread A runs the compiled backward, which calls the
  compiled FX graph. The FX graph calls into Python, which eventually calls
  `call_cpp_tensor_pre_hooks`. Meanwhile Thread B triggers a different code
  path that also calls `call_cpp_tensor_pre_hooks` (unlikely but possible
  since it is a publicly exported method). Thread B reads `active_rstate`
  which is either `nullptr` (assert fires) or points to Thread A's
  `RuntimeState` (data race on the `RuntimeState` internals). More
  realistically, if `compiled_autograd` releases the GIL during FX graph
  execution (it does not currently, but `pybind11::gil_scoped_acquire` is at
  function scope), another thread could call `call_cpp_tensor_pre_hooks`.
  The write (pointer store) and read are both on a plain (non-atomic) pointer
  -- this is a C++ data race (undefined behavior).
- **Suggested fix:** Make `active_rstate` thread-local, or pass the
  `RuntimeState` as a Python capsule argument rather than using a global.
