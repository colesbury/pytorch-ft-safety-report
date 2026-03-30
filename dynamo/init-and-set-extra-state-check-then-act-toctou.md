# `init_and_set_extra_state` check-then-act TOCTOU

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** The "extra" slot on the code object (accessed via
  `_PyCode_GetExtra` / `_PyCode_SetExtra`).
- **Writer(s):** `init_and_set_extra_state()` -- checks
  `get_extra_state(code) == nullptr`, then allocates and sets.
  Called from: `dynamo__custom_eval_frame` (line 505-506),
  `dynamo_set_code_exec_strategy` (line 689-690),
  `dynamo_skip_code_recursive` (line 702-703),
  `_load_precompile_entry` (line 277-278).
- **Reader(s):** `get_extra_state()` in every caller.
- **Race scenario:** Thread A and Thread B both call the same function for
  the first time. Both see `extra == nullptr`. Both call
  `init_and_set_extra_state()`. The CHECK inside will fire on the second
  thread (since the first already set it), causing an assertion failure /
  abort. Even if the CHECK is removed, the second `_PyCode_SetExtra` call
  will overwrite the first's pointer, and `destroy_extra_state` will be
  called on the first allocation, leading to use-after-free if Thread A
  already has a pointer to the first ExtraState.
- **Tier:** **Tier 2** (requires two threads hitting the same un-compiled
  function simultaneously).
- **Suggested fix:** Use a compare-and-swap or a global lock around
  init_and_set_extra_state. Alternatively, use `_PyCode_GetExtra` +
  `_PyCode_SetExtra` atomically (CPython 3.14t may need internal locking on
  code extras for this). A simple approach: use `std::call_once` or a
  per-code-object lock.
