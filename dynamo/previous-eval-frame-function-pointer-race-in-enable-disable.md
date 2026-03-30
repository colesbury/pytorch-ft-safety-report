# `previous_eval_frame` function pointer race in enable/disable

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `previous_eval_frame` -- a file-scope function pointer
  (in `eval_frame.c`, line 187).
- **Writer(s):**
  - `enable_eval_frame_shim()` -- reads the current eval frame func, saves
    it in `previous_eval_frame`, and installs the dynamo shim.
  - `enable_eval_frame_default()` -- restores `previous_eval_frame` and sets
    it to NULL.
- **Reader(s):**
  - `dynamo_eval_frame_default()` -- reads `previous_eval_frame` to decide
    whether to call it or `_PyEval_EvalFrameDefault`.
- **Race scenario:** Thread A calls `set_eval_frame(callback)` which calls
  `increment_working_threads` -> `enable_eval_frame_shim`. Thread B
  simultaneously calls `set_eval_frame(None)` which calls
  `decrement_working_threads` -> `enable_eval_frame_default`. The
  read-modify-write on `previous_eval_frame` is not atomic.
  Additionally, `_PyInterpreterState_SetEvalFrameFunc` sets the eval frame
  function on the **interpreter** (shared by all threads), not per-thread.
  If Thread B sets it back to `previous_eval_frame` while Thread A is still
  using the shim on a different core, Thread A's in-flight frame evaluation
  could see inconsistent state.
- **Tier:** **Tier 2** (requires concurrent enable/disable of dynamo).
  Could also be **Tier 1** if the main thread enables dynamo while another
  thread is already executing frames.
- **Suggested fix:** Protect enable/disable with a mutex. The
  `active_dynamo_threads` counter is also not atomic and has the same
  TOCTOU issues.
