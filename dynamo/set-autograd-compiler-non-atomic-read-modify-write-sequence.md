# `set_autograd_compiler` non-atomic read-modify-write sequence

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 2 (requires concurrent `set_autograd_compiler` calls)
- **Shared state:** `the_autograd_compiler`, `default_dyn_type_int`,
  `Engine::the_compiled_autograd` (atomic in engine.cpp).
- **Writer(s):** `set_autograd_compiler()` performs a multi-step sequence:
  reads `the_autograd_compiler` (line 1261), reads `default_dyn_type_int`
  (line 1262), writes `default_dyn_type_int` (line 1263), conditionally writes
  `the_autograd_compiler` (lines 1265/1269), and calls
  `Engine::set_compiled_autograd` (lines 1266/1270). No lock is held.
- **Reader(s):** Another concurrent `set_autograd_compiler` call, or the
  compiled backward path.
- **Race scenario:** Two threads call `set_autograd_compiler` concurrently.
  Thread A reads `prior_compiler = the_autograd_compiler`, capturing a pointer.
  Thread B overwrites `the_autograd_compiler` and `Py_INCREF`s its new value.
  Thread A then overwrites with its own value. The prior captured by Thread A
  is stale -- it may point to an object that Thread B has already returned for
  decref. The refcount bookkeeping becomes corrupted, leading to either a leak
  or a double-free.
- **Suggested fix:** Protect the entire `set_autograd_compiler` function body
  with the compiled autograd mutex.
